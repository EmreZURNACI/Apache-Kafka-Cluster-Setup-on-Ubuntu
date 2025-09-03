# Apache Kafka Cluster Setup on Ubuntu

This document provides a step-by-step guide for setting up an Apache Kafka cluster on Ubuntu.  
The instructions can be applied to a single machine for testing or scaled across multiple servers for a production environment.


# Steps
1. Update the package list:

    ```bash
        sudo apt update
    ```

2. Install Java Runtime Environment (JRE):

    ```bash
        sudo apt install default-jre -y
    ```

3. Download and extract Apache Kafka (choose the binary version you want, here we use **3.9.1**):

    ```bash
        wget https://dlcdn.apache.org/kafka/3.9.1/kafka_2.13-3.9.1.tgz
    ```

4. Extract the downloaded archive:

    ```bash
        tar -zxvf kafka_2.13-3.9.1.tgz
    ```

5. Rename the extracted folder to kafka: 

    ```bash
        mv kafka_2.13-3.9.1 kafka
    ```

6. Create a directory to store logs from Kafka and Zookeeper services: 

    ```bash
        mkdir -p /home/ubuntu/kafka/serviceLogs
    ```

7. Instead of manually starting and stopping Zookeeper with `./bin/zookeeper-server-start.sh` every time, create a **systemd service** to manage it.: 

    ```bash
        sudo vim /etc/systemd/system/zookeeper.service
    ```

    ```bash
        [Unit]
        Description = Zookeeper Service
        Requires = network.target remote-fs.target
        After = network.target remote-fs.target

        [Service]
        User = root
        Type = simple
        ExecStart = /bin/sh -c "/home/ubuntu/kafka/bin/zookeeper-server-start.sh /home/ubuntu/kafka/config/zookeeper.properties > /home/ubuntu/kafka/serviceLogs/zookeeper.log"
        ExecStop = /bin/sh -c "/home/ubuntu/kafka/bin/zookeeper-server-stop.sh"
        Restart = on-abnormal

        [Install]
        WantedBy = multi-user.target
    ```

8.  In the same directory, create a **systemd service** for the Kafka broker as well.

    ```bash
        sudo vim /etc/systemd/system/kafka.service
    ```

    ```bash
        [Unit]
        Description = Kafka Service
        Requires = zookeeper.service
        After = zookeeper.service

        [Service]
        User = root
        Type = simple
        ExecStart = /bin/sh -c "/home/ubuntu/kafka/bin/kafka-server-start.sh /home/ubuntu/kafka/config/server.properties > /home/ubuntu/kafka/serviceLogs/kafka.log"
        ExecStop = /bin/sh -c "/home/ubuntu/kafka/bin/kafka-server-stop.sh"
        Restart = on-abnormal

        [Install]
        WantedBy = multi-user.target
    ```

9.  Register the new services:

    ```bash
        sudo systemctl daemon-reload
    ```

10. Set the services to start automatically on every machine boot and start them:

    ```bash
        sudo systemctl enable zookeeper
        sudo systemctl enable kafka
    ```

    ```bash
        sudo systemctl start zookeeper
        sudo systemctl start kafka
    ```

ℹ️ **Tip:** Your basic stand-alone Kafka broker is now ready.You can use the `./bin` directory to run any Kafka commands or tasks you need..

11. Set ownership for the `/data` directories so that Zookeeper and Kafka can write to them:

    ```bash
        sudo mkdir -p /data/kafka
        sudo mkdir -p /data/zookeeper
    ```

    ```bash
        sudo chown -R ubuntu:ubuntu /data/kafka
        sudo chown -R ubuntu:ubuntu /data/zookeeper
    ```

12. Assign a unique ID to each Zookeeper instance on every machine:

    ```bash
        echo "1" > /data/zookeeper/myid
    ```

13. Configure Zookeeper in the `zookeeper.properties` file:

    I.	Open the configuration file:

    ```bash
       vim /home/ubuntu/kafka/config/zookeeper.properties
    ```

    II.  Change the data directory:

    ```bash
       dataDir=/data/zookeeper
    ```
	
    III.  Set additional Zookeeper limits required for cluster stability: 
	
	```bash
		initLimit=5
		syncLimit=2
	```

	IV.  Add all Zookeeper nodes at the end of the file in the format `server.X=[hostname]:2888:3888` :
	
	```bash
        server.1=machine1.example.com:2888:3888
        server.2=machine2.example.com:2888:3888
        server.3=machine3.example.com:2888:3888
    ```

14. Configure Broker in the `server.properties` file:

    I.  Open the configuration file:

    ```bash
       vim /home/ubuntu/kafka/config/server.properties
    ```

    II.  Assign a unique broker ID for each Kafka node:

    ```bash
       broker.id=1
    ```

    III.  Set the listeners and advertised listeners to the machine's own IP address:

    ```bash
		listeners=PLAINTEXT://0.0.0.0:9092
        advertised.listeners=PLAINTEXT://machine1.example.com:9092
    ```

    IV.  Change the log directory:

    ```bash
        log.dirs=/data/kafka
    ```

    V.  Connect the broker to all Zookeeper nodes:

    ```bash
        zookeeper.connect=machine1.example.com:2181,machine2.example.com:2181,machine3.example.com:2181
    ```

15. Start using the services by restarting them sequentially on all machines:

    ```bash
        sudo systemctl restart zookeeper
        sudo systemctl restart kafka
    ```
