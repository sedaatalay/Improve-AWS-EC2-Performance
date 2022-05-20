# Juice up your EC2 using Jumbo Frames - Maximum Transmission Unit (MTU)

The maximum transmission unit (MTU) of a network connection is the size, in bytes, of the largest allowed packet that can be passed over the link. The larger the MTU of a link, the more data can be transmitted in a single packet. Ethernet packets consist of the frame, or the actual data you're sending, and the network overhead information surrounding it.

To take advantage of this feature and improve your application performance, you can configure your applications to use larger frameworks. All Amazon EC2 instance types support 1500 MTUs, and many existing instance sizes support 9001 MTU or jumbo frames.

![improve-ec2-performance-with-jumbo-frames](https://user-images.githubusercontent.com/91700155/169384870-ecd20210-cf8c-403e-b545-2d4ae3ace678.png)


How can we test if our new 9000-byte MTU is actually working, and do we take advantage of a larger packet size? We can use tracepath to determine the maximum MTU between two endpoints.

```console
[ssh -i h_pair.pem ubuntu@52.29.193.171 -L localhost:8037:localhost:8080]$ tracepath google.com
1?: [LOCALHOST]                                         pmtu 9001
1:  ip-10-13-0-1.ec2.internal                             0.226ms pmtu 1500
1:  no reply
:
5:  no reply
6:  100.65.14.17                                          0.807ms asymm  7 
7:  52.93.29.63                                          31.645ms asymm  8 
8:  100.100.2.36                                          0.738ms asymm 12 
9:  99.82.181.25                                          0.845ms asymm 16 
10:  no reply
:
30:  no reply
    Too many hops: pmtu 1500
    Resume: pmtu 1500 
```
    
   In the above route(to the internet) the maximum MTU was 1500, While i do the same with EC2 instances in the same subnet, i am able to achieve a maximum of MTU of 9001

```console
[ssh -i h_pair.pem ubuntu@52.29.193.171 -L localhost:8037:localhost:8080]$ tracepath 10.13.0.126
1?: [LOCALHOST]                                         pmtu 9001
:
30:  no reply
    Too many hops: pmtu 9001
    Resume: pmtu 9001 
```
    
   **Can we really use this in a real-world scenario?**

   Let us assume two high I/O intensive EC2 workloads are there in the same subnet. They are processing huge files (~10GBs) between them. We will copy the files between the instances and count how many the _Incoming Packets_ are sent when `MTU 1500` is used and how many are needed when `MTU 9001` is set.
    
### Check the MTU Size

Lets check the current MTU 

```console
# Set it to to 1500
sudo ip link set dev eth0 mtu 1500
ip link show eth0
```

### Create a huge file
    
```console
# Assuming you have 20GB free space
fallocate -l 20G /home/ubuntu/test.img
```

### Send some data

   You can use `scp` or `rsync` to copy the data. I have already copied the private pem(`h_pair.pem`) to allow me to securely copy the data between the machines

```console
pcksFile="/sys/class/net/eth0/statistics/rx_packets"
nbPcks=`cat $pcksFile`
# scp -i h_pair.pem ubuntu@52.29.193.171 -L localhost:8037:localhost:8080:/home/ubuntu/test.img /home/ubuntu/
rsync -a --info=progress2 \
   -e "ssh -i h_pair.pem" \
   ec2-user@10.13.0.126:/home/ubuntu/test.img \
   /home/ubuntu/
echo $(expr `cat $pcksFile` - $nbPcks)
```

   *Repeat previous step with mtu set to `9001`*

```console
sudo ip link set dev eth0 mtu 9001
ip link show eth0
pcksFile="/sys/class/net/eth0/statistics/rx_packets"
nbPcks=`cat $pcksFile`
# scp -i h_pair.pem ubuntu@52.29.193.171 -L localhost:8037:localhost:8080:/home/ec2-user/test.img /home/ubuntu/
rsync -a --info=progress2 \
   -e "ssh -i h_pair.pem" \
   ubuntu@52.29.193.171 -L localhost:8037:localhost:8080:/home/ubuntu/test.img \
   /home/ubuntu/
echo 'Total Incoming Packets: $(expr `cat $pcksFile` - $nbPcks)'
```
  _Output_: My experiments with a 20GB file show that, it take about `14860928`packets with MTU 1500 and `2418970`packets with MTU 9001. That is close to 6x improvement.
  
  
### Conclusion

   As you can see, you can get almost 6x performance improvements if you configure your applications to make use of the higher `MTU` offered to by AWS and get more out of your EC2 instances.


<p></br>
  
Thank you :)

