# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **4883**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

<!-- How did you create the tunnel loop (which entry, on which switch, which commands)?
     What did the switch logs show over time, with numbers from loop-before.txt?
     Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     loop in this program? Check the code, and explain what happens when TTL reaches 0. -->
The forwarding loop was created by modifying the MyIngress.myTunnel_exact table entry for dst_id = 3 on switch s2. Instead of forwarding the packet towards its destination, s2 was configured to send it back towards s1.

The command using is sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"

This assumes table handle 2 corresponds to dst_id = 3, as confirmed in the earlier table output. The output port was changed to 1, creating a forwarding path from s1 to s2 and back to s1.

The logs in loop-before.txt showed that both switches continued processing packets. The packet-processing counts increased over time, indicating that the packet was repeatedly circulating between the switches instead of being dropped.

No. In tunnel.p4, ipv4_forward only subtracts 1 from the TTL. It never tests for zero and never calls drop(). P4 arithmetic wraps at the field’s width, so an 8-bit TTL at 0 becomes 255 and keeps going. An IPv4 loop would therefore not be stopped either. I confirmed this by reading the code, not by running an IPv4 loop
## 2. Design: the hop limit

<!-- Your header change and your pipeline change, and the reason for each choice:
     - the field's size, and where in the header you put it (and why there);
     - the starting value send.py uses, and how you chose it;
     - where it's decremented, and what happens when it runs out;
     - what else had to change for every part of the path to agree (parser, deparser, hosts). -->

I added an 8-bit hop_limit field to the custom tunnel header:

An 8-bit field supports values from 0 to 255, which is sufficient for limiting the number of tunnel-switch forwarding operations. The sender uses an initial value of 5:

h1 python3 send.py 10.83.0.2 "test 1" --tunnel 2 --hops 5

The forwarding action decrements the hop limit whenever a tunnel packet is forwarded:
action myTunnel_forward(egressSpec_t port) {
    standard_metadata.egress_spec = port;
    hdr.myTunnel.hop_limit = hdr.myTunnel.hop_limit - 1;
}


Before applying the tunnel forwarding table, the ingress pipeline checks whether the hop limit is zero:
if (hdr.myTunnel.isValid()) {
    if (hdr.myTunnel.hop_limit == 0) {
        drop();
    } else {
        myTunnel_exact.apply();
    }
}

This means a packet with a hop limit of zero is dropped instead of being forwarded again. A packet with a positive value can continue, but its hop limit is decreased by one.

Other parts of the packet-processing path also needed to be updated:
- Parser: Extracts the updated 5-byte tunnel header.
- Deparser: Emits the updated tunnel header.
- Sender: Adds the hop_limit field and allows the initial value to be configured using --hops.
- Receiver: Uses the updated tunnel header format so that it can decode and display the remaining hop limit.
The sender, switches and receiver must agree on the header format. Otherwise, the packet fields may be interpreted at incorrect byte offsets.

## 3. Implementation

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->
tunnel.p4 -> 	Added the 8-bit hop_limit field, decremented it during tunnel forwarding, and dropped packets when the incoming value was zero.
send.py -> Added the --hops argument, with a default value of 5, and included the value in the transmitted tunnel header
receive.py -> Updated the receiver's tunnel-header definition and packet decoding to display the remaining hop limit.

I kept tunnel_v1.p4 unchanged as the original implementation. This allowed me to test a mixed deployment where one switch runs the new program and another continues running the old program.
## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->
The tunnel delivery tests confirmed that the updated header could be transmitted and decoded correctly.
The packet captures for h2 and h3 showed the updated tunnel header and confirmed that the hop-limit byte was in the expected position. The captured value was 03 at the destination.

The loop-after test also showed that the packet-processing counts stopped increasing after the hop-limit protection was enabled. This indicates that the loop was no longer continuously forwarding packets.

For a packet sent with an initial hop limit of 5, the packet passed through two forwarding switches:

mininet> h1 ping -c 2 h3
PING 10.83.0.3 (10.83.0.3) 56(84) bytes of data.
64 bytes from 10.83.0.3: icmp_seq=1 ttl=62 time=11.2 ms
64 bytes from 10.83.0.3: icmp_seq=2 ttl=62 time=10.2 ms

By design the packet is forwarded 5 times and dropped on the 6th sighting: 6 packets handled in total, 3 per switch, with the drop happening at s2. After the drop the counts stopped rising.


## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->
Mixed rollout 
Prediction. The old s2 expects IPv4 immediately after a 4-byte tunnel header. With the new 5-byte header it would start reading IPv4 one byte too early. Because it has no hop_limit field, it would also never decrement it.

• s1 received 05 from the sender and forwarded 04. The capture at h2 still shows 04, not 03, so s2 did not decrement it.
• The BMv2 log for s2 shows the 4-byte tunnel header extracted, then IPv4 extracted starting at the hop-limit byte. Every IPv4 field was therefore read one byte off.
• The packet was still forwarded. The lookup uses dst_id at bytes 2–3, the same offset in both programs. The old tunnel path never uses or changes the misparsed IPv4 fields. The deparser then wrote out exactly the bytes it had parsed, so the packet left unchanged.

From the design I expect the loop to still end, but later: s1 decrements on every sighting and s2 never does, so s1 handles 6 packets and s2 handles 5. 

A safe way to roll out a header change

The mixed test shows the worst case for a header change: the old switch did not drop the packet or complain, it misparsed it and forwarded it. A safe rollout should make old switches fail closed (drop the packet) and should not send the new format until every switch on the path can read it. I compared three options.

1. A new EtherType for the new format. An old switch’s parser does not recognise the new value, so it ends with no valid headers and its ingress drops the packet. The failure is loud and safe. New switches parse both EtherTypes. The downside: while some switches are still old, new-format traffic through them is dropped, so a path cannot mix old and new switches.
2. A version field in the header. A new switch can serve several formats, and future changes become easier. But old switches never look at it, so it does not protect them, and it only helps if it is in the format from the start. Adding one later is the same problem again.
3. Placing the new field where the old parser ignores it. Appending the byte after dst_id fails this test, as Section 5a shows: the old parser expects IPv4 straight after 4 bytes.

Hop limit (data plane). It acts on every real packet at the moment it is forwarded and needs no controller. It catches loops from any cause: the one in C1, which was created by a direct run-time table_modify that no controller validated, as well as races during updates, failed links and stale entries. Its limits: it only bounds the damage (a looping packet still uses up to 5 hops of bandwidth), it does not say where or why the loop formed, it protects only tunnelled packets (the IPv4 path is still unprotected), and it does nothing about non-looping errors such as blackholes.
check_paths() (control plane, Week 3). It examines the intended forwarding state before or when it is installed. It can reject a loop before any packet flows, name the entries that form the cycle, and check whole-network properties such as reachability. 

Cost on a line-rate hardware switch
• Bandwidth. One extra byte per tunnelled packet: about 1.6% of a 64-byte frame and about 0.07% of a 1500-byte frame. A maximum-size frame could also exceed the MTU once the header grows.
• Parser. One more byte to extract and one more 8-bit field to carry through the pipeline.
• Match–action. One 8-bit comparison and one 8-bit subtraction: simple ALU operations that need no new table and no TCAM or SRAM entries. The if creates a dependency, because the table lookup depends on its outcome. Dependencies cost stages, so in a tight pipeline this could use one extra stage. With a spare stage it costs nothing in throughput. No CPU involvement or recirculation is needed.
• Deployment. The change needs a reprogrammable parser, which a fixed-function chip cannot offer. The compatibility cost shown in Section 5a is probably larger than the silicon cost.

## AI use

I used ChatGPT to explain P4 and BMv2 concepts, help interpret logs and packet captures, discuss the hop-limit design, and draft and refine this report. I ran every command and produced all evidence files myself, and I checked each claim in this report against my own outputs. I can explain and defend every part. [Edit this so it matches what really happened.]

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
