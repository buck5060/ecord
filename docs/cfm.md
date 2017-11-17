# Configuring CFM
This guide describes how to configure CFM monitoring for an E-Line generated in a local E-CORD POD. 
This monitoring will then be viewable in the “Delay and Jitter” link in the CORD UI. 
It assumes that your POD already has a global E-CORD node and at least 2 local E-CORD instances, with each local E-CORD having a CFM capable device (e.g. Microsemi EA1000) connected.  
It is assumed that these devices are already detected and connected in ONOS as per [https://guide.opencord.org/profiles/ecord/installation_guide.html](the guide).

The guide acceses an ONOS_CORD instance at onos-cord, which has the ssh port 8102 and the web port 8182. Change the IP adddresses to your own, and if these ports have been forwarded for some reason on your setup, make sure to accommodate as necessary.

This guide describes a minimal CFM setup, with a single MD (maintenance domain) and MA (maintenance association), two MEPs (measurement endpoints), and one DM (delay measurement).

You can also find a [https://www.getpostman.com/](Postman) JSON collection that contains the JSON necessary to setup CFM under `cord/orchestration/profiles/ecord/examples/`.

Detailed information on the CFM application itself can be found on the [https://wiki.onosproject.org/display/ONOS/Layer+2+Monitoring+with+CFM+and+Services+OAM](ONOS Wiki).
## Creating an MD
Post the following JSON file to `onos-cord:8182/onos/cfm/md/` on your local instances. Make sure you have a header with basic authorization set to username “onos” and password “rocks”.

NOTE: MDs and MAs are stored in a distributed store within the same cluster. In our testing setup, ONOS_CORD and ONOS_Fabric exist on two separate clusters. If you see the MD on the ONOS_Fabric (i.e. the MD shows up after running cfm-md-list) after instantiating it on the first instance, do not push the JSON on the second nodd. Otherwise, feel free to push.

If you are creating more than one MD, they must have different mdNumericIds. Be aware that devices will only be able to take delay measurements from those on the same MD.

```json
{"md": {
    "mdName": "Microsemi",
    "mdNameType": "CHARACTERSTRING",
    "mdLevel": "LEVEL3",
    "mdNumericId": 1
   }
}
```
## Creating an MA
The MAs created should be a part of a pre-defined MD. We defined an MD called Microsemi, so we post the following JSON to `onos-cord:8182/onos/cfm/md/Microsemi/`.

NOTE: MAs are also stored in a distributed store. See the previous note in “Setting up MDs” for possible consequences.

If you are creating more than one MA, they must have different maNumericIds and vids. The `rmep-list` defines the list of MEP IDs that can be associated with this vlan, so if you need more than 2 MEPs you will need to add them to the `rmep-list` .

```json
{
  "ma": {
    "maName": "ma-vlan-1",
    "maNameType": "CHARACTERSTRING",
    "maNumericId": 1,
    "ccm-interval": "INTERVAL_1S",
    "component-list": [
      { "component": {
        "component-id":"1",
        "tag-type": "VLAN_STAG",
        "vid-list": [
          {"vid":1}
        ]
        }
      }
    ],
    "rmep-list": [
      { "rmep":10 },
      { "rmep":20 }
    ]
  }
}
```
## Creating a MEP
A measurement endpoint (MEP) is associated with a Microsemi device, and one must be created for each one. A MEP can only be created for the device connected to the current local node, and must be associated with an MA. The MEP ID must already exist on that MA’s `rmep-list` and must not already be in use by another node.

With our MD set to Microsemi, and MA set to ma-vlan-1, we post the following JSON to `onos-cord:8182/onos/cfm/md/Microsemi/ma/ma-vlan-1/mep`

```json
{
  "mep": {
    "mepId": 10,
    "deviceId": "netconf:10.6.0.160:830",
    "port": 0,
    "direction": "DOWN_MEP",
    "mdName": "Microsemi",
    "maName": "ma-vlan-1",
    "primary-vid": 1,
    "administrative-state": true,
    "ccm-ltm-priority": 4,
    "cci-enabled" :true
  }
}
```

Since we have 2 Microsemi devices, we will also need to post MEP creation JSON to our second local instance, making sure to change the `mepId` and `deviceId`. In our example, we defined two MEP IDs, 10 and 20, to be used with our first and second local nodes. Both MEPs must be set up correctly in order to create a delay measurement (DM).

## Creating a DM
Check the `remote_mep_state` of your created meps to ensure that they have a value of `RMEP_OK` before creating delay measurements. You can do so by performing a GET; in our case, we would perform a GET on `onos-cord:8182/onos/cfm/md/Microsemi/ma/ma-vlan-1/mep/10` on our first local instance, and on `onos-cord:8182/onos/cfm/md/Microsemi/ma/ma-vlan-1/mep/20` on our second local instance.

DMs are associated with a single MEP, and a maximum of 2 can exist on a single MEP.

IMPORTANT: The `measurementsEnabled` in the sample JSON below are required for the CFM stat UI to function properly.

To create a DM on our first local instance, the following JSON is submitted by POST to `onos-cord:8182/onos/cfm/md/Microsemi/ma/ma-vlan-1/mep/10`. Notice that the remoteMepId is set to 20 because we are trying to get the delay measurement to 20 from 10.

```json
{
  "dm": {
        "remoteMepId":20,
        "dmCfgType": "DMDMM",
        "version": "Y17312008",
        "priority": "PRIO1",
        "messagePeriodMs": 100,
        "startTime": {
          "immediate":true
        },
        "stopTime": {
          "none":true
        },
        "frameSize": 1500,
        "measurementIntervalMins": 1,
        "measurementsEnabled": [
            "INTER_FRAME_DELAY_VARIATION_TWO_WAY_AVERAGE",
            "INTER_FRAME_DELAY_VARIATION_TWO_WAY_MAX",
            "INTER_FRAME_DELAY_VARIATION_TWO_WAY_MIN",
            "INTER_FRAME_DELAY_VARIATION_TWO_WAY_BINS",
            "FRAME_DELAY_TWO_WAY_AVERAGE",
            "FRAME_DELAY_TWO_WAY_MAX",
            "FRAME_DELAY_TWO_WAY_MIN",
            "FRAME_DELAY_TWO_WAY_BINS"
        ]
  }
}
```

We don’t need to create a DM on the second local instance for this DM to work. Make sure that you aren’t setting the `remoteMepId` equal to the mepId you are trying to create the DM on.

## Viewing Delay and Jitter

At this point, we are done configuring CFM and can now view delay and jitter statistics in the CORD UI under “Delay and Jitter”. Login into the CORD UI at `onos-cord/xos` with the username `xosadmin@opencord.org`, and the password generated in the local node under `/opt/credentials/xosadmin@opencord.org`. The "Delay and Jitter" link will be on the left navigation panel once you have successfully logged in.