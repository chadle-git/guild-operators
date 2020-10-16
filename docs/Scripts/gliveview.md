!> Ensure the [Pre-Requisites](basics.md#pre-requisites) are in place before you proceed.

**Guild LiveView - gLiveView** is a utility to display an equivalent subset of LiveView interface that cardano-node users have grown accustomed to. This is especially useful when moving to a systemd deployment - if you haven't done so already - while looking for a familiar UI to monitor the node status.

The tool is independent from other files and can run as a standalone utility that can be stopped/started without affecting the status of cardano-node.

##### Download

If you've used [prereqs.sh](basics.md#pre-requisites), you can skip this part , as this is already set up for you. The tool rely on the common env configuration file.  
To get current epoch blocks, [cntoolsBlockCollector.sh](Scripts/cntools-blocks.md) script is needed. This is optional and **Guild LiveView** will function without it.

?> For those who follow guild's [folder structure](basics.md#folder-structure) and do not wish to run prereqs.sh, you can run the below in `$CNODE_HOME/scripts` folder

To download the script:

```bash
curl -s -o gLiveView.sh https://raw.githubusercontent.com/cardano-community/guild-operators/master/scripts/cnode-helper-scripts/gLiveView.sh
curl -s -o env https://raw.githubusercontent.com/cardano-community/guild-operators/master/scripts/cnode-helper-scripts/env
chmod 755 gLiveView.sh
```

##### Configuration & Startup

For most setups, it's enough to set `CNODE_PORT` in the `env` file. The rest of the variables should automatically be detected. If required, modify User Variables in `env` and `gLiveView.sh` to suit your environment. 

For most standard deployments, this should lead you to a stage where you can now start running `./gLiveView.sh` in the folder you downloaded the script (default location for cntools users would be `$CNODE_HOME/scripts`). Note that the script is smart enough to automatically detect when you're running as a Core or Relay and will show fields accordingly.

The tool can be run in legacy mode with only standard ASCII characters for terminals with trouble displaying the box-drawing characters. Run `./gLiveView.sh -h` to show available command-line parameters or permanently set it directly in script.

A sample output from both core and relay(with peer analysis):

![Core](https://raw.githubusercontent.com/cardano-community/guild-operators/images/gliveview-core.png)

![Relay](https://raw.githubusercontent.com/cardano-community/guild-operators/images/gliveview-relay.png)

##### Description

**Upper main section**  
Displays live metrics gathered from EKG. Epoch number and progress is live from node while date calculation until epoch boundary is based on offline genesis parameters. Reference tip is also an offline calculation based on genesis values used to compare against the node tip to see how far of the tip(diff value) the node is. With current parameters a slot diff up to 40 from reference tip is considered good but it ussually stay below 30. In/Out peers show how many connections the node have established in and out.

**Core section**  
If the node is run as a core, identified by the 'forge-about-to-lead' EKG parameter, a second core section is displayed. This section contain current and remaining KES periods as well as a calculated date for the expiration. When getting close to expire date the values will change color. Blocks created by the node since node start is another metric shown in this section. If [cntoolsBlockCollector.sh](Scripts/cntools-blocks.md) is running for the core node a second row with current epoch blocks is displayed.

**Peer analysis**  
A manual peer analysis can be triggered by key press `p`. A latency test will be done on incoming and outgoing connections to the node. For outgoing connections a normal ICMP ping is done as a first try. If this is blocked, tcptraceroute program is used to do a tcp ping against the cardano-node port of the remote peer. For incoming connections only ICMP ping is used as remote peer port is unknown. It's not uncommon to see many unreachable peers for incoming connections as it's a good security practice to disable ICMP in firewall.

Once the analysis is finished, it will display the RTTs for the peers and group them in ranges 0-50, 50-100, 100-200, 200<. The analysis filters out multiple connections to the same IP. This means each unique IP address will only be pinged once, and subsequent peers from the same IP address will be skipped. Unique + Skipped should match the total number seen in top section. The analysis is **NOT** live. Last update timestamp is shown to know when it was last run. Press `p` for update or `h` to hide it.

##### Troubleshooting/Customisations

In case you run into trouble while running the script, you might want to edit `env` & `gLiveView.sh` and look at User Variables section shown below. You can override the values if the automatic detection do not provide the right information, but we would appreciate if you could also notify us by raising an issue against github repo:

**env**
```
######################################
# User Variables - Change as desired #
# Leave as is if unsure              #
######################################

#CCLI="${HOME}/.cabal/bin/cardano-cli"                  # Override automatic detection of path to cardano-cli executable
#CNODE_HOME="/opt/cardano/cnode"                        # Override default CNODE_HOME path (defaults to /opt/cardano/cnode)
CNODE_PORT=6000                                         # Set node port
#CONFIG="${CNODE_HOME}/files/config.json"               # Override automatic detection of node config path
#SOCKET="${CNODE_HOME}/sockets/node0.socket"            # Override automatic detection of path to socket
#EKG_HOST=127.0.0.1                                     # Set node EKG host
#EKG_PORT=12788                                         # Override automatic detection of node EKG port
#EKG_TIMEOUT=3                                          # Maximum time in seconds that you allow EKG request to take before aborting (node metrics)
#BLOCK_LOG_DIR="${CNODE_HOME}/db/blocks"                # CNTools Block Collector block dir set in cntools.config, override path if enabled and using non standard path
#CURL_TIMEOUT=10                                        # Maximum time in seconds that you allow curl file download to take before aborting (GitHub update process)
```

**gLiveView.sh**
```bash
######################################
# User Variables - Change as desired #
######################################

NODE_NAME="Cardano Node"                   # Change your node's name prefix here, keep at or below 19 characters!
REFRESH_RATE=2                             # How often (in seconds) to refresh the view (additional time for processing and output may slow it down)
LEGACY_MODE=false                          # (true|false) If enabled unicode box-drawing characters will be replaced by standard ASCII characters
RETRIES=3                                  # How many attempts to connect to running Cardano node before erroring out and quitting
THEME="dark"                               # dark  = suited for terminals with a dark background
                                           # light = suited for terminals with a bright background
```