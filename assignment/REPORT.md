# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **2598**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

     1) How did you create the tunnel loop (which entry, on which switch, which commands)?
     The tunnel loop was created by changing entry 2 in the MyIngress.myTunnel_exact table on switch s2.The original forwarding port for dst_id = 3 was changed to port 1 using command (sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"). This caused packets to be forwarded back towards s1 instead of reaching h3, creating a forwarding loop between s1 and s2.

     2) What did the switch logs show over time, with numbers from loop-before.txt?
     After sending a tunnelled packet into the loop, the switch logs continued recording packet processing. The initial log counts 1536 for s1 and 1534 for s2. After 5 seconds, the counts changed to 516 for s1 and 514 for s2.

     3) Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     loop in this program? Check the code, and explain what happens when TTL reaches 0. 
     The IPv4 forwarding action decreases the TTL by 1 whenever a packet is forwarded.However, it does not check whether the TTL has reached 0 or drop the packet when it reaches 0. Therefore, it does not properly stop an IPv4 forwarding loop. Since TTL is an 8 bit field, decreasing it below 0 can cause it to wrap around to 255 instead of automatically dropping the packet. An explicit TTL check is needed to stop the loop

## 2. Design: the hop limit
   
     - An 8 bit field called hop_limit was added after dst_id in the tunnel header. It only adds 1 byte to the header and can store values from 0 to 255.
     - In send.py, the default starting value is set to 8. This value allows packets to pass through several switches while preventing them from being forwarded forever.
     - The myTunnel_forward action in tunnel.p4 was updated to decrease the hop limit by 1 whenever a packet is forwarded. If the value is already 0, the packet will be dropped to prevent it from continuing in a loop.
     - The parser and deparser did not need major changes because they already extract and emit the complete tunnel header. They can handle the additional field automatically.
     - The header definitions in both send.py and receive.py were also updated with the new hop_limit field. This allows the sender to include the field and the receiver to display its value correctly.
     

## 3. Implementation

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->
     i) tunnel.p4
     The tunnel header was extended by adding an 8 bit hop_limit field after the existing dst_id field. The myTunnel_forward action was updated to check the hop limit before forwarding a packet. If the value is zero, the packet is dropped using mark_to_drop(). Otherwise, the hop limit is decreased by 1 before the packet is forwarded. This prevents tunnelled packets from circulating indefinitely.

     ii) send.py
     The send.py file was modified to support the new hop limit field. A --hops option was added so the sender can set the starting hop limit, with a default value of 8. This value is included in the tunnel header whenever a tunnelled packet is sent. For packets sent without tunnelling, the original behaviour remains unchanged.

     iii) receive.py
     The receiver was updated to recognise the new 8-bit hop_limit field in the tunnel header. It now displays the destination ID and remaining hop limit when a tunnelled packet arrives. This makes it possible to observe how the hop limit changes during packet forwarding.

## 4. Evidence: it works

 Show, with your evidence files:
     i) tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
        The tunnelled packet was successfully delivered to both h2 and h3. Both receivers displayed a hop limit of 6, compared to the initial value of 8. This shows that the hop limit decreased by two during forwarding. (Evidence: assignment/evidence/delivery.txt)

     ii) plain IPv4 still works (delivery.txt);
        Normal IPv4 communication was tested using h1 ping -c 3 h3. All three packets were successfully received with 0% packet loss. (Evidence: assignment/evidence/delivery.txt)

     iii)  the same loop as in section 1 now ends, and after how many passes — does that match your design? (loop-after.txt) 
         The tunnel loop was recreated by changing the forwarding entry on s2. After sending a packet into the loop, the switch log counts remained unchanged after 3 seconds: s1 stayed at 723 and s2 stayed at 719. This indicates that the loop stopped. (Evidence: assignment/evidence/loop-after.txt)
     
         The design allows up to eight forwarding operations before the packet is dropped on the next forwarding attempt when its hop limit is zero. The stable log counts support that the loop stopped, although the exact number of forwarding passes was not directly captured. (Evidence: assignment/evidence/loop-after.txt)

## 5. Evaluation

    a)  i) In the mixed-version rollout, s1 runs the updated tunnel.p4 and s2 runs the old tunnel_v1.p4. Packet  delivery might fail because the new tunnel header has an additional 1byte hop_limit field while the old s2 program only expects the original 4 byte tunnel header. The old s2 program would read the extra hop_limit byte as the beginning of the IPv4 header, causing incorrect IPv4 parsing.
    
    Also, s2 would not decrease the hop limit because it does not recognise the new field. However, s1 can still decrease the hop limit when the packet returns to it and eventually drop the packet.

        ii) The packet was not received by h2 during the mixed-version delivery test. The packet capture showed 0 packets captured.From the s2 log, the old program read dst_id = 2 and forwarded the packet through port 2. However, it did not understand or process the new hop_limit field.

        For the loop test, the switch log counts remained unchanged after 3 seconds. s1 stayed at 733 and s2 stayed at 728. This shows that the loop stopped even though s2 was running the old program.

        iii) The result was mostly as expected. The new tunnel header has 5 bytes, but the old s2 program only reads 4 bytes. This means the old program does not recognise the extra hop_limit field. The extra byte (08) is read as the start of the IPv4 header, instead of the actual IPv4 starting byte (45). Because of this, the packet is not read correctly and h2 does not receive it.

        For the loop test, the result was also as expected. Although s2 does not know about the hop limit, s1 can still decrease it when the packet returns. The log counts stayed at 733 for s1 and 728 for s2 after 3 seconds, showing that the loop had stopped.


     b)   One way is to add a version field to the tunnel header. This helps new switches know which header  format they are receiving and makes it easier to support both old and new versions.Another way is to use a new EtherType for the new tunnel format. This helps old switches recognise that the packet uses a different format instead of reading it as the old one.

     Personally, I think using a version field is a good choice because switches can easily identify different header versions. The network can also be upgraded step by step instead of changing all switches at once.

     c) The hop limit works in the data plane. It limits how many times a packet can be forwarded and stops packets from going around in a loop for too long.check_paths() works in the control plane. It checks the network paths and helps detect possible loops in the network configuration before packets are sent.

     Real networks use both because check_paths() helps prevent loops before they happen while the hop limit provides extra protection when packets are travelling through the network.

     d) Adding the hop limit increases the tunnel header size by 1 byte.The switch needs to check the hop limit, decrease its value and drop the packet if it reaches zero which  requires a small amount of extra processing.It also uses a small amount of additional hardware resources. The actual performance cost depends on the switch hardware and its ability to process packets at line rate.
        


## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->

     ChatGPT was used to provide guidance on modifying and understanding the code in tunnel.p4, send.py, receive.py, and topo.py. It assisted with implementing the hop-limit mechanism, making changes to send.py and receive.py, troubleshooting coding errors, understanding the network topology, interpreting BMv2 switch logs, and verifying the implementation through testing in the Mininet environment.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
