# P4 Tables, Match and Actions

## Understanding the P4 API

Fundamentally, P4 exposes an API on the forwarding hardware which a control plane can interact with.  Look inside the JSON config files for each switch `./exercises/basic/pod-topo/sX-runtime.json` [here](./exercises/basic/pod-topo/) - you will see control plane definitions of the match-action tables and their entries/rows.  For example, consider this match-action entry in the table `MyIngress.ipv4_forward`:

~~~
"table_entries": [
    ...
    {
      "table": "MyIngress.ipv4_lpm",
      "match": {
        "hdr.ipv4.dstAddr": ["10.0.1.1", 32]
      },
      "action_name": "MyIngress.ipv4_forward",
      "action_params": {
        "dstAddr": "08:00:00:00:01:11",
        "port": 1
      }
    },
    ...
]
~~~

You will find this the corresponding action and table defined in 
[basic.p4](./exercises/basic/basic.p4) as:

~~~
    /* Definition of action "ipv4_forward". 
    It is akin to a function definition.  Note how this action 
    takes the parameters 'dstAddr' and 'port', which are passed from 
    the control plane in the "action_params" dict in "sX-runtime.json".  
    Inside the action, packet fields are modified, such as decrementing 
    the ttl counter. 

    'standard_metadata' is a set of fields in the V1Model and it is 
    implementation specific, but in this case BMV2 implements the V1Model. 
    A comprehensive list of the fields can be found in p4-cheat-sheet.pdf.
    */

    action ipv4_forward(macAddr_t dstAddr, egressSpec_t port) {
        standard_metadata.egress_spec = port;
        hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
        hdr.ethernet.dstAddr = dstAddr;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }

    /* Definition of table "ipv4_lpm". 
    The "key" object defines the matching criteria: 
    "<header_field>:<match_type>". The "actions" object contains a
    list of actions (defined above) that are applied the packets in
    order of preference.
    */

    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm;
        }
        actions = {
            ipv4_forward;
            drop;
            NoAction;
        }
        size = 1024;
        default_action = drop();
    }
~~~

So you see that the `*.p4` file defines tables, match criteria and 
actions and exposes an API on the forwarding device to the control plane. 
The control plane can then interact with this API to populate the 
match-action tables within the device.
