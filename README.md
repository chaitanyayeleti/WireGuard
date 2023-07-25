# WireGuard VPN Congiguration 

### Accessing the VPN Gateway using SSH
  - To acesses newly created server over SSH from windows you can use CMD or terminal 
     ``` bash
    ssh -p 22 username@publicip 
    ```
  - "Perform a repository update and upgrade, and then proceed to install WireGuard. By using '&&' between commands, it ensures that each command runs sequentially, one after the other."
    ``` bash
    sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get install wireguard -y
    ```
  ### Configuring WireGuard
  - "Within WireGuard, you'll find two command-line tools: wg and wg-quick, which serve the purpose of configuring and managing the WireGuard interfaces."
  - Run the following command to generate the public and private keys:
    
    ``` bash
    wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
    ```
  - Use cat or less cmd to copy private & public key never share private key with anyone .
  - With the keys successfully generated, the next step is to configure the tunnel device responsible for routing the VPN traffic. You have two options to set up the device:

    1. Command-line setup: Utilize the `ip` and `wg` commands to configure the device directly through the command line.
    
    2. Configuration file setup: Alternatively, create a configuration file using a text editor to define the tunnel device's settings.
    3. Create a new file named wg0.conf and add the following contents:
    
       ``` bash
       sudo nano /etc/wireguard/wg0.conf
       ```

       ``` bash
       [Interface]
        Address = 10.0.0.1/24
        SaveConfig = true
        ListenPort = 51820  # VPN Gateway NSG we already opened 
        PrivateKey = SERVER_PRIVATE_KEY
        PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE # eth0 is the defualt interface for azure linux VM if you have any other change it .
        PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o etho -j MASQUERADE
       ```
    - The wg0.conf and privatekey files should not be readable to normal users. Use chmod to set the permissions to 600:
      ``` bash
      sudo chmod 600 /etc/wireguard/{privatekey,wg0.conf}
      ```
    - After completing the configuration, bring up the wg0 interface using the attributes specified in the configuration file. You can do this by running the appropriate command, which might look like:
      ``` bash
      sudo wg-quick up wg0
      ```
    - This command will activate the wg0 WireGuard interface with the settings defined in the configuration file, allowing it to start routing VPN traffic as configured.
   
    - The command will produce an output similar to the following:
      ``` bash
      [#] ip link add wg0 type wireguard
      [#] wg setconf wg0 /dev/fd/63
      [#] ip -4 address add 10.0.0.1/24 dev wg0
      [#] ip link set mtu 1420 up dev wg0
      [#] iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
      ```
    - Run wg show wg0 to check the interface state and configuration:
      ``` bash
      sudo wg show wg0
      ```
      ```
      interface: wg0
      public key: r3imyh3MCYggaZACmkx+CxlD6uAmICI8pe/PGq8+qCg=
      private key: (hidden)
      listening port: 51820
      ```
    - To bring the WireGuard interface at boot time run the following command:
      ``` bash
      sudo systemctl enable wg-quick@wg0
      ```
    - For NAT to work, we need to enable IP forwarding. Open the /etc/sysctl.conf file and add or uncomment the following line:
      ``` bash
      sudo nano /etc/sysctl.conf
      ```
      - uncomment & Change the below value to 1
      ```
      net.ipv4.ip_forward=1
      ```
      - Save the file and apply the change:
        ``` bash
        sudo sysctl -p
        ```
      - Output
        ``` bash
        net.ipv4.ip_forward = 1
        ```
      - By defualt azure linux VM doesnt have ufw enabled if you want to use enable and allow all the ports wwhich we enabled on NSG
    ## END of WireGuard Configuration on Server level


    ## Client level Configuration (Windows,Linux,MacOS or andriod)
      # For MacOS & Linux configuration is same
      - Install wireguard using same CMD which i mentioned ealier
      - Generate Public Key & Private key using same cmd 
      - Create  config file using the same CMD how we created on server and past below details
        ``` bash
        [Interface]
        PrivateKey = CLIENT_PRIVATE_KEY # Copy client private key and paste here
        Address = 10.0.0.2/24   # I prefer to give static IP . use the same ip range from the server config
        DNS = 1.1.1.1
        
        
        [Peer]
        PublicKey = SERVER_PUBLIC_KEY # Server Public Key
        Endpoint = SERVER_IP_ADDRESS:51820  # Server Static IP and udp Port
        AllowedIPs = 0.0.0.0/0
        ```
      - Save the Config before enabling add the peer into server below is the cmd need to execute in Server
        ``` bash
        sudo wg set wg0 peer CLIENT_PUBLIC_KEY allowed-ips 10.0.0.2  # Client Public Key and static ip which we configured on client config
        ```
      - On Linux or MacOs clients run the following command the bring up the interface:
        ``` bash
        sudo wg-quick up wg0
        ```
      - You can verify the connection using  **sudo wg**  cmd on both Client & Server
    
        

        

      
       
       




