

<div align="center">
  <h1>FlyNet: Simulation Platform for Flying Ad-hoc Networks</h1>

  <img src="https://img.shields.io/badge/Contribution-Welcome-yellowgreen" height="20">
  <img src="https://img.shields.io/badge/License-MIT-brightgreen" height="20">

  <h3>Make simulation more friendly to novices with high-fidelity modeling! </h3>
</div>

This Python-based simulation platform can realistically model various components of the UAV network, including the network layer, MAC layer and physical layer, as well as the UAV mobility model, energy model, etc. In addition, the platform can be easily extended to meet the needs of different users and develop their own protocols.

## Requiremens
- Python >= 3.3 
- Simpy >= 4.1.1
  
## Installation and usage
Firstly, download this project:
```
git clone https://github.com/ZihaoZhouSCUT/Simulation-Platform-for-UAV-network.git
```
You can even ```run main.py``` directly with one click to get a sneak peek. But we recommend that you go to ```utils/config.py``` to browse and roughly understand the simulation parameters before starting the operation, including: the size of the simulation map, the number of drones, the velocities of drones, the routing protocol adopted and so on.

## Project structure
<div align="center">
<img src="https://github.com/itsmert/Wireless-Network-UAV-Simulator-Adhoc" width="800px">
</div>

## Core logic
The following figure shows the main procedure of packets transmissions in FlyNet. "Drone's buffer" is a resource in SimPy whose capacity is one, which means that drone can send at most one packet at a time. If there are many packets need to be transmitted, they need to queue for buffer resources according to the time order of arrival to the drone. We can simulate the queuing delay by this mechanism. Besides, we note that there are two other containers: "transmitting_queue" and "waiting_list". For all the "data packets" and "control packets" generated by drone itself or received from other drones but need to further forward, the drone will firstly put them into "transmitting_queue". A function called "feed_packet" will periodically read the packet at the head of the "transmitting_queue" every very short time, and let it to wait for the "buffer" resource. It should be noted that "ACK packet" wait for buffer resource directly without being put into the "transmitting_queue".

After the packet is read, a packet type determination will be performed first. If this packet is control packet (usually no need to decide the next hop), then it will directly start to wait for buffer resource. When this packet is data packet, next hop selection will be executed by routing protocol, if an appropriate next hop can be found, then this data packet can start wait for buffer resource, otherwise, this data packet will be put into "waiting_list" (mainly in reactive routing protocol now). Once the drone has the relevant routing information, it will take this data packet from "waiting_list" and add it back to "transmitting_queue".

When packet gets the buffer resource, MAC protocol will be performed to access the wireless channel. When the packet is successfully received by the next hop, packet type determination needs to be performed. For example, if the received packet is data packet, an ACK packet is needed to reply after SIFS. In addition, if the receiver is the destination of the incoming data packet, some metrics will be recorded (PDR, end-to-end delay, etc.), otherwise, it means that this data packet needs to be further relayed so it will be put into the "transmitting_queue" of the receiver drone.


## Module overview
### Routing protocol
In this project, **Greedy Perimeter Stateless Routing (GPSR)**, **Gradient Routing (GRAd)**, **Destination-Sequenced Distance Vector routing (DSDV)** and some **Reinforcement-Learning based Routing Protocol** have been implemented. The following figure illustrates the routing procedure of GRAd and GPSR. More detail information can be found in the corresponding papers.



### Media access control (MAC) protocol
In this project, **Carrier-sense multiple access with collision avoidance (CSMA/CA)** and **Pure aloha** have been implemented. I will give a brief overview of the version implemented in this project, and focus on how signal interference and collision are implemented in this project. The following picture shows the example of packets transmission when CSMA/CA (without RTS/CTS) protocol is adopted. When a drone wants to transmit packet:

1. it first needs to wait until the channel is idle
2. when the channel is idle, the drone starts a timer and waits for "DIFS+backoff" periods of time, where the length of backoff is related to the number of re-transmissions
3. if the entire decrement of the timer to 0 is not interrupted, then the drone can occupy the channel and start sending the packet
4. if the countdown is interrupted, it means that the drone loses the game. The drone then freeze the timer and wait for channel idle again before re-starting its timer



The following figure demonstrates the packets transmission flow when pure aloha is adopted. When a drone installed a pure aloha protocol wants to transmit packet:

1. it just sends it, without listening to the channel and random backoff
2. after sending the packet, the node starts to wait for the ACK packet
3. if it receives ACK in time, the "mac_send" process will finish
4. if not, the node will wait a random amount of time, according to the number of re-transmissions attempts, and then sends packet again



From the above illustration we can see that, it is not only two drones sending packets at the same time that cause packet collisions. If there is an overlap in the transmission time of two data packets, it means that collision occurrs. So in our project, each drone checks its inbox every very short interval and has several important things to do (as shown in the following figure):

1. delete the packet records in its inbox whose distance from the current time is greater than twice the maximum packet transmission delay. This reduces computational overhead because these packets are guaranteed to have already been processed and will not interfere with packets that have not yet been processed
2. check the packet records in the inbox to see which packet has been transmitted in its entirety
3. if there is such a record, then find other packets that overlap with this packet in transmission time in the inbox records of all drones, and use them to calculate SINR.



### Mobility model
Mobility model is one of the most important mudules to show the characteristics of UAV network more realistically. In this project, **Gauss-Markov 3D mobility model**, **Random Walk 3D mobility model** and **Random Waypoint 3D mobility model** have been implemented. Specifically, since it is quite difficult to achieve continuous movement of drone in discrete time simulation, we set a "position_update_interval" to update the positions of drones periodically, that is, it is assumed that the drone moves continuously within this time interval. If the time interval "position_update_interval" is smaller, the simulation accuracy will be higher, but the corresponding simulation time will be longer. There will be a trade-off. Besides, the time interval that drone updates its direction can also be set manually. The trajectories of a single drone within 100 second of the simulation under the two mobility models are shown as follows:



### Motion control module



### Energy model
The energy model of our platform is based on the work of Y. Zeng, et al. The figure below shows the power required for different drone flying speeds. The energy consumption is equal to the power multiplied by the flight time at this speed.


## Performance evaluation
Our "FlyNet" platform supports the evaluation of several performance metrics, as follows:

- **Packet Delivery Ratio (PDR)**: PDR is the ratio of the total number of received data packets successfully at all destination drones over the total number of data packets generated by all source drones. It should be noted that PDR excludes redundant data packets. PDR can reflect the reliability of the routing protocol.
- **Average End to End Delay (E2E Delay)**: E2E delay is the average time delay for data packets to reach from the source drone to the destination drone. Typically, delay in packet transmission involves "queuing delay", "access delay", "transmission delay", "propagation delay (So small as to be negligible)" and "processing delay".
- **Normalized Routing Load (NRL)**: NRL is the ratio of all routing control packets send by all drones to number of received data packets at the destination drones.
- **Average Throughput**: In our platform, the calculation of throughput is: whenever the destination receives a packet, the length of the packet is divided by the end-to-end delay of the packet (because E2E delay involves the re-transmissions of this data packet)
- **Hop Count**: Hop count is the number of router output ports through which the packet should pass.

## Design your own protocol
Our simulation platform can be expanded based on your research needs, including designing your own mobility model of drones (in ```mobility``` folder), mac protocol (in ```mac``` floder), routing protocol (in ```routing``` floder) and so on. Next, we take routing protocols as an example to introduce how users can design their own algorithms.

 * Create a new package under the ```routing``` folder (Don't forget to add ```__init__.py```)
 * The main program of the routing protocol must contain the function: ```def next_hop_selection(self, packet)``` and ```def packet_reception(self, packet, src_drone_id)```
 * After confirming that the code logic is correct, you can import the module you designed in ```drone.py``` and install the routing module on the drone:
   ```python
   from routing.gpsr.gpsr import Gpsr  # import your module
   ...
   class Drone:
     def __init__(self, env, node_id, coords, speed, certain_channel, simulator):
       ...
       self.routing_protocol = Gpsr(self.simulator, self)  # install
       ...
   ```

## Contributing
Issues and pull requests are warmly welcome! 

## Show your support
Give a ⭐ is this project helped you!

## License
This project is MIT licensed.
