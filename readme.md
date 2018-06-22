# ShadowP4 compiler

## Features

- ShadowP4 compiler can merge two P4 programs to one fully automated. The target is Bmv2.
- Support A-B testing. The traffic for A-B testing is configurable at runtime through flow entry in STC table. 
- Support Differential testing, as described in the paper.

## Merge two P4 programs
The following is the commands to merge two P4 program:
```
usage: ShadowP4c-bmv2.py [-h] [--real_source source]
                         [--shadow_source SHADOW_SOURCE] [--json_s JSON_S]
                         [--json_mg JSON_MG] [--gen_dir GEN_DIR] [--json JSON]
                         [--gen-fig] [--version] [--primitives PRIMITIVES]

ShadowP4 compiler bmv2 (optional) arguments:
  -h, --help            show this help message and exit
  --real_source source  A source file to include in the P4 program.
  --shadow_source SHADOW_SOURCE
                        A shadow P4 program source file to merge.
  --json_s JSON_S       Dump the JSON representation to shadow P4 file.
  --json_mg JSON_MG     Dump the JSON representation to merged P4 file.
  --gen_dir GEN_DIR     The dir to store the generated graphs datas.
  --json JSON           Dump the JSON representation to production file.
  --gen-fig             The dir for the generated shadow configure files and graphs.
  --version, -v         show program's version number and exit
  --primitives PRIMITIVES
                        A JSON file which contains additional primitive
                        declarations
  --AB-testing           Merging for A-B Testing case
  --Diff-testing         Merging for Differential Testing case
```

For example, we can compiler the AB testing demo by the following command, the merged output json file for bmv2 is `switch_merged.json`:
```
python ShadowP4c-bmv2.py --real_source         tests/Demo/merge-simple-AB/switch_prod.p4 \
                         --shadow_source       tests/Demo/merge-simple-AB/switch_test.p4 \
                         --json_mg             tests/Demo/merge-simple-AB/switch_merged.json \
                         --gen_dir             tests/Demo/merge-simple-AB \
                         --AB-testing
```


### Shadow Traffic Control
STC is used to manage the traffic for both testing program and production program. STC support four action current:

1) Action `SP4_add_shadow_tag`: turn the production to testing traffic
2) Action `SP4_remove_shadow_tag`: turn the testing traffic to production traffic
3) Action `goto_testing_pipe`: send the packet to testing pipeline
4) Action `goto_production_pipe`: send the packet to production pipeline

Each flow entry in STC table is associated with one action. All the traffic matched the flow will be processed by the action. The default match fields of the flow entry is dest mac address. Operators can custom the match fields in the file `SP_metas`, at line 104 `reads` fields, the following gives a example for the match fields:
```
    reads {
        ethernet.dstAddr : exact;
    }
```

By changing the flow entry in STC, operators can management the shadow traffic at run time. For example, the following command can guide all the packets matched to testing pipeline. 
```
table_add shadow_traffic_control goto_testing_pipe 00:04:00:00:00:01 =>
``` 

### Shadow configure management and agent
ShadowP4 compiler will generate a ShadowP4 configure file in the generated directory, name `ShadowP4Configure`. SPM can translate the control messages, such as the flow entry add messages. More detail are in `SPM` directory.



### Visualization merging
We can also generate the visible graph of merged parser and control flow using `--gen-fig` parameter. The figures will be generated in the specific directory.



### Differential testing

- Merge two P4 programs

The following command example shows how to merge two P4 programs for Differential Testing:
```
python ShadowP4c-bmv2.py --real_source   tests/Demo/merge-DiffTesting/router_prod.p4 \
                        --shadow_source       tests/Demo/merge-DiffTesting/router_test.p4 \
                        --json                tests/Demo/merge-DiffTesting/router_prod.json \
                        --json_s              tests/Demo/merge-DiffTesting/router_test.json \
                        --json_mg             tests/Demo/merge-DiffTesting/router_merged.json \
                        --gen_dir             tests/Demo/merge-DiffTesting \
                        --gen-fig \
                        --Diff-testing
```

- Run the merged P4 program

We can get the merged json file for bmv2 in the `gen_dir`. Here is an example to run the merged program in mininet.
```
sudo python tools/create_1sw_mininet.py --behavioral-exe /usr/local/bin/simple_switch --num-host 4 --json tests/Demo/merge-DiffTesting/router_merged.json
```

- Runtime populate production/testing tables

In the merged P4 program, there are three kinds of table can be configured through runtime CLI: 1) production program tables, 2) testing program tables and 3) shadow traffic control tables.

For the Differential testing case, we give three files in `tests/Demo/merge-DiffTesting` as the detailed examples of run-time configure commands.

    - `commands_prod.txt` : run-time example commands for production program

    - `commands_test.txt`: run-time example commands for testing program

    - `commands_STC.txt`: run-time example commands for shadow traffic control. 

Note that, as those entries are used for the single P4 programs before merging, operates should translate them into entries adapted for the merged P4 program before configuring them according to `ShadowP4Configure`. SPM can make it with:
```
python SPM/SPM_translate_cmd.py -cfg tests/Demo/merge-DiffTesting/ShadowP4Configure -c tests/Demo/merge-DiffTesting/commands_test.txt -n tests/Demo/merge-DiffTesting/commands_test_new.txt
```

Next we can install those new commands.
```
sudo python tools/runtime_CLI.py < tests/Demo/merge-DiffTesting/commands_test_new.txt 
```

- Runtime configure STC tables and testing error handles
For the STC, we set packets with destination L2 address `00:04:00:00:00:01` to testing packets.

For the testing error handles, we can configure the `handle_comparator_error` table to forward the error packet to controller.


## Advanced features

### Reconfigure the STC muodule in both cases
The shadow traffic module is reconfigurable. We can set the match fields of the STC tables `shadow_traffic_control` for the shadow traffic classification.

### Reconfigure the comaprator fields in Differential testing
The comparator fields is reconfigurable through two actions in the meta file `SP4_metas_xx`. We can set any fields in packet header or metadata. The two actions to record the output of P4 programs are in the following.
```
/* record running result of production P4 program */
action _rcd_production_result() {
    modify_field(shadow_metadata.meta_p, standard_metadata.egress_spec);
}

/* record running result of shadow testing P4 program */
action _rcd_shadow_result() {
    modify_field(shadow_metadata.meta_t, standard_metadata.egress_spec);
}
```

### Reconfigure the detected error procedure in Differential testing
It's also defined in the meta file `SP4_metas_xx`. The action to handle the error packet ia `handle_comparator_error`.


### Beyond the A-B testing and Differential testing
Todos.


## To improve

- all the parser state should start from the `start` + `parse_ethernet`.
- todo: test the action selector and action profile features.