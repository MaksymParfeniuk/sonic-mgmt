

### Repository structure

```
sonic-mgmt/
├── ansible/                  # Testbed definitions, inventory, deployment playbooks
│   ├── testbed.yaml          #   physical testbeds
│   ├── vtestbed.yaml         #   virtual (KVM) testbeds
│   ├── veos, veos_vtb, lab   #   inventory files (hosts, credentials)
│   ├── testbed-cli.sh        #   add/remove topology, deploy-mg
│   ├── vars/topo_*.yml       #   topology definitions (t0, t1, ptf32 ...)
│   └── group_vars/, host_vars/
├── tests/                    # pytest tests
│   ├── conftest.py           #   global options, core fixtures, pytest_plugins
│   ├── run_tests.sh          #   wrapper that builds the pytest command
│   ├── test_pretest.py, test_posttest.py   # sanity checks before/after a run
│   ├── common/
│   │   ├── devices/          #   SonicHost, MultiAsicSonicHost, DutHosts classes
│   │   ├── fixtures/         #   shared fixtures
│   │   ├── helpers/          #   plain helper functions
│   │   ├── plugins/          #   pytest plugins 
│   │   └── testbed.py        #   testbed file parsing
│   ├── bgp/, pfcwd/, acl/, qos/, ...   # test areas, one folder per feature
│   ├── ptftests/, saitests/  #   PTF traffic scripts
│   ├── vtestbed.yaml, veos_vtb          # virtual testbed files used by the test run
│   └── scripts/, templates/
├── docs/
│   ├── testbed/              #   testbed setup guides
└   └── tests/                #   test documentation
```



### Ansible 

`ansible` This folder contains codes for SONiC testbed deployment and setup. 
	`vtestbed.yaml` - contains different testbed definition.
	`veos`, `veos_vtb`,etc - inventory files, host lists for DUTs, servers and VMs.
	`group_vars`, `host_vars` - contains variables for each group and host such as credentials and connection details.
	`vars/` - directory full of topologies (t0, t1, t2, etc).
	`testbed-cli.sh` - the main script for different topologies operation. For example adding and removing topologies, deploying topology .
	`templates/` - config templates.
	`testbed_add_vm_topology.yml` - playbook that are used by `testbed-cli.sh` to start the topology.
	`testbed_remove_vm_topology.yml` - playbook that are used by `testbed-cli.sh` to remove the existing topology.
	`testbed_start_VMs.yml` - playbook to start VMs used by testbed-cli.sh
	`config_sonic_basedon_testbed.yml` and `deploy_config_on_testbed.yml` - playbooks that push configuration to the DUT.




##### vars/
This directory contains a lot of different topologies such as t0, t1 and t2.
Each topology file contains the following structure.
```yaml
topology:
  host_interfaces:            # DUT ports connected to the PTF
    - 0
    - 1
  disabled_host_interfaces:   # host ports that are left disabled
    - 0
  VMs:                        # neighbour VMs connected to the DUT
    ARISTA01T1:
      vlans:                  # DUT port(s) this VM connects to
        - 28
      vm_offset: 0            # index of the VM within the testbed's VM range
  DUT:                        # settings for the DUT itself
    vlan_configs: ...

configuration_properties:     # shared properties such as common BGP/AS values
  common: ...

configuration:                # per-device config: hostname -> BGP, interfaces and IPs
  ARISTA01T1:
    properties: [...]
    bgp: {asn: ..., peers: ...}
    interfaces: {Loopback0: ..., Ethernet1: ...}
    bp_interface: ...
```

##### vtestbed.yaml 
This is the testbed definition file for the virtual testbed

The fields of each entry:
	`conf-name` and  `group-name` - the testbed name and its group
	`topo` - the topology name which refers to a file in `ansible/vars/topo_*.yml`
	`ptf_image_name`, `ptf`, `ptf_ip` - the PTF container
	`server`- the server that runs the topology.
	`vm_base` - the first VM name of neighbour VMs.
	`dut` - the device under test hostname. Can be more than one.
	`inv_name` - the inventory file name to use. Refers to `ansible/*` . For example if our inv_name is veos_vtb we refer to ansible/veos_vtb file


##### testbed-cli.sh
This script are using ansible playbooks to operate with topologies.

the main operations:
`start-vms` / `stop-vms <server>` - used `testbed_start_VMs.yml`and `testbed_stop_VMs.yml` to start or stop the neighbour VMs on a server.
`add_topo` - used `testbed_add_vm_topology.yml` to deploy topology for specified testbed on a server. 
`remove_topo` - used `testbed_remove_vm_topology.yml` to remove topology for specified testbed on a server.
`deploy_mg` - used `config_sonic_basedon_testbed.yml` to deploy minigraph to DUT.




### Tests

`tests` This folder contains all tests and running test script

`conftest.py`  - The central pytest file. It holds CLI options, `tbinfo`, `duthosts`, `ptfhost`, the `enum_*` fixtures and `pytest_generate_tests`.
`pytest.ini`  - Config for the run.
`run_tests.sh` - The wrapper script around pytest.
`test_pretest.py`, `test_posttest.py`They prepare the DUT before a run and clean up and collect diagnostics after it.
`ansible.cfg`, `vtestbed.yaml`, `veos_vtb`, `vtestbed.csv` - Ansible config and virtual-testbed files used when running from `tests/`.
`ptf_runner.py` - Runs PTF scripts on the PTF host.

Test areas (one folder per feature): `bgp/`, `acl/`, `qos/`, `pfcwd/`, `route/`, `vlan/`, `fdb/`, and about 100 more. A test is a `test_*.py` file inside its feature folder.

`common/`- the shared library.
- `devices/`- the host classes (`SonicHost`, `MultiAsicSonicHost`, `DutHosts`, `PTF`).
- `fixtures/`- shared fixtures.
- `helpers/` - plain helper functions.
- `plugins/` - pytest plugins.
`ptftests/`, `saitests/`, `acstests/` - PTF scripts that send and receive packets on the PTF host. They are not pytest tests.
`scripts/` - helper scripts that get copied to the PTF or the DUT.

`./run_tests.sh`
This script is a wrapper around pytest.
It builds the right `pytest` command lines and runs them in three stages.
1. First `setup_environment`
	Sets default variables
	Exports ansible variables
	kills any running pytest and ansible-playbook processes.
2. Option parsing `getopts`
	Parse the options passed through the `./run_tests.sh` script.
3. Builds the pytest command `setup_test_options`
	It builds the list of test files and common options.

Example ./run_tests.sh command
`./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c bgp/test_bgp_fact.py -f vtestbed.yaml -i ../ansible/veos_vtb -u`
(`-d`, `-n` are mandatory; also one of `-t` or `-c`. `-u` skips the pretest/posttest group.)


### Docs

holds the project documentation for sonic-mgmt. It explains how to set up testbeds, how to write and run tests, and what each test is meant to verify.

Its main subfolders:

- `docs/testbed/`: guides for building and configuring testbeds
- `docs/tests/`: guides for writing, organising and running pytest tests, including logging, style rules and parallel runs.
- `docs/ansible/`: documentation for the Ansible deployment and test code.
- `docs/api_wiki/`: reference for the methods of the host objects (`duthost`, `ptfhost`, `localhost`) that tests call.
- `docs/testplan/`: test plans that describe what each feature test should verify such as BGP, ACL.

### Test bgp facts 

Path to test: `sonic-mgmt/tests/bgp/test_bgp_fact.py`
The test is a thin wrapper and the main logic is in `run_bgp_facts` function in `tests/common/helpers/bgp.py`

how the DUT is selected and accessed
`run_tests.sh -d <dut>` passes the selected DUT to pytest as `--host-pattern` and `-n <testbed>` specifies the testbed. `pytest_generate_tests` checks if the test uses `enum_frontend_dut_hostname` and creates a separate test instance for each frontend DUT (and ASIC if `enum_asic_index` is used). If a specific DUT is selected, only that DUT is used.  

We can access DUT by passing enum_fronted_dut_hostname as a index of duthosts like this 
`duthosts[enum_frontend_dut_hostname]` 
after that we can retrieve the corresponding `MultiAsicSonicHost` object (`duthost`)

###### What it tests.
The tests checks if the DUT bgp state matches the configured state. For each ASIC of each DUT.
- Sessions are up: each neighbour in `bgp_facts` is `established`
- local ASN is correct.
- STATE_DB agrees: `NEIGH_STATE_TABLE|<nbr>` shows `Established` and the right `peerType` (`i-BGP` if the remote and local ASN are equal, otherwise `e-BGP`). It polls with `wait_until(30, 5, 0, ...)` because `bgpmon` updates this table with a lag.
- config matches runtime: compare the bgp neighbors name with config db bgp neighbor name. Also compare the bgp neighbors ASN with config db.

 **PASS**: every neighbor is `established`, local AS correct, STATE_DB `NEIGH_STATE_TABLE` reaches `Established` + correct `peerType` within 30 s, names/ASNs match CONFIG_DB.
 **FAIL**: any `assert` in `run_bgp_facts` fails, also fixture errors.
 **SKIP**: only via markers - `pytest.mark.topology('any','t0-sonic','t1-multi-asic')` (skipped if `--topology` doesn't match), `pytest.mark.device_type('vs')` (skipped if `--device_type` differs).

##### fixtures dependency diagram for the bgp facts test

![[Pasted image 20261123115531.png]]


Fixture Scopes
Scopes - define how long a fixture is reused
Available scopes: `function` (per test), `class` (per test class), `module` (per Python file), `package` (per package), `session` (once per pytest run)

Autouse
autouse=True – makes a fixture run automatically without explicitly adding it to the test arguments. For example, `enhance_inventory` in `conftest.py`.

Yield
yield – separates fixture execution into two parts: setup (before `yield`) and teardown (after `yield`). 
##### Fixture explanation
`test_bgp_facts` uses 3 fixtures: `duthosts`, `enum_asic_index` and `enum_frontend_dut_hostname`

#### duthosts
`duthosts`
The fixture contains `DUTs` hosts defined in `testbed`.
Also it uses `enhance_inventory`, `ansible_adhoc`, `tbinfo`, `request`, `ipv6_only_mgmt_enabled` fixtures.

to collect DUT hosts it uses `DutHosts` class
`DutHosts`
This is the container of all DUTs. It behave like a list: it supports `duthosts[0] `and `duthosts['sonic1']`
defined here: `tests/common/devices/duthosts.py`
Contains 3 attributes:
`nodes`: list of all `MultiAsicSonicHost` instances
`frontend_nodes`: subset of nodes (DUTs with front-panel ports)
`supervisor_nodes`: subset of nodes (supervisor DUTs)


`MultiAsicSonicHost` 
This class represents one SONiC device `duthost`,Multi-asic `SonicHost`. 
Defined here: `tests/common/devices/multi_asic.py`
it has two attributes:
`sonic_host`: a `SonicHost` instance.
`asics`: a list of `SonicAsic` instance.


`SonicHost`
This class represents a SONiC device and provides SONiC-specific methods.
Defined here: `tests/common/devices/sonic.py`
It has two main responsibilities:
- Collects and stores device facts (`facts`), such as the SONiC version and ASIC information.
- Provides methods to work with the device, for example checking services, Docker containers, interfaces, and BGP

It inherits from `AnsibleHostBase`, so it can execute Ansible modules through the base class.

`SonicAsic`
This class represents one ASIC inside a SONiC device.
Defined here: `tests/common/devices/sonic_asic.py`
It has attributes:
`sonichost`: reference to the parent `SonicHost`.
`hostname`: DUT hostname.
`asic_index`: ASIC number.
`ns_arg`: namespace argument used when executing commands.

Its main purpose is to execute commands and Ansible modules in the correct ASIC namespace. This allows tests to work with a specific ASIC without manually specifying namespace arguments.

`AnsibleHostBase`
This is the base class for executing Ansible modules on a host.
Defined here: `tests/common/devices/base.py`
The responsibilities if it. it is Initializes the Ansible connection to the specified host using `ansible_adhoc`.

##### enhance_inventory

The `enhance_inventory` fixture allows passing multiple inventory files separated by commas instead of just one. If `trim_inv` is enabled, it also trims the inventory to keep only the required DUTs, removing unnecessary hosts.

##### ansible_adhoc
it returns `init_host_mgr`, which initializes and returns the Ansible Host Manager using the current pytest configuration. allows us to access DUTs defined in the inventory.
It returns a function that can be used to obtain an Ansible host object for a specific DUT, through which we can execute Ansible modules such as `shell`, `command`, and `bgp_facts`

##### tbinfo
Calls `get_tbinfo` to obtain testbed information.


`get_tbinfo`
This is the helper function to create and return testbed information.
it requests `--testbed` and `--testbed_file` from config.
and calls `TestbedInfo` class

`TestbedInfo`
This class contains a lot of helper methods to parse the testbed file.
Returns `TestbedInfo` object
parsed data stored in `tbinfo.testbed_topo` in OrderedDict format
Example:
```
OrderedDict({
    "t0": {
        "group-name": "t0",
        "topo": {
            "name": "t0",
            "type": "t0",
            "properties": {...},
            "ptf_map": {...},
            "ptf_map_disabled": {...},
            "ptf_dut_intf_map": {...}
        },
        "ptf_image_name": "...",
        "ptf": "...",
        "ptf_ip": "192.168.0.1",
        "ptf_netmask": "255.255.255.0",
        "duts": ["dut1", "dut2"],
        "duts_map": {"dut1": 0, "dut2": 1},
        ...
    }
})
```

##### ipv6_only_mgmt_enabled
Fixture that check and configures ipv6 management mode.
if --ipv6_only_mgmt or -6 is passed to pytest 
this enables ipv6-only management mode where DUT mgmt_ip will use the ipv6-address.

#### enum_asic_index
`enum_asic_index`
returns an identifier/index of asic
for a single-Asic DUT it uses the repository default ASIC_ID `DEFAULT_ASIC_ID`
for a multi-Asic DUT it uses the inventory ASICS_PRESENT value if it is available
if no it makes indices from 0 up to the configuread ASIC count.

#### enum_frontend_dut_hostname

fixture that gives a test the hostname of frontend DUT
returns hostname string per test case

### `test_bgp_facts` lifecycle


Before pytest: `run_tests.sh`
It parses the options, kills stale pytest and Ansible processes. Then it runs the pretest stage, the tests, and the posttest stage.

Startup (`tests/conftest.py`)
`pytest_addoption` registers `--testbed`, `--testbed_file`, `--inventory`, `--host-pattern` and the others that we've put into `./run_tests.sh`.
`pytest_configure` and `pytest_sessionstart` set up the session.

Collection and parametrization
`pytest_generate_tests` checks which fixtures the test uses and generates test instances based on the testbed configuration. For example, it creates separate instances for different DUTs and ASICs.

Fixture setup
Pytest initializes the required fixtures according to their scope.
`duthosts` creates a `DutHosts` object, which contains `MultiAsicSonicHost` instances for the selected DUTs. During initialization, it connects to the DUTs through Ansible and gathers device facts.

Other autouse fixtures and plugins also perform their setup here.

Test execution
The `test_bgp_facts` test compares BGP information from FRR, STATE_DB and CONFIG_DB. It only reads information from the DUT.

Teardown and report
After the test finishes, pytest tears down the fixtures in reverse order of their scope.

Lifecycle summary
`run_tests.sh` (env, args) -> pretest (`test_pretest.py`) -> pytest start (`pytest_addoption`, `pytest_configure`, `pytest_sessionstart`) -> collection + `pytest_generate_tests` + marker skips -> session fixtures (`tbinfo`, `duthosts`) -> module fixtures (`enum_*`, `setup`) -> function fixtures -> test body -> teardown after `yield` in reverse order -> report/junit -> posttest (`test_posttest.py`).


### Source references (tests/)

| Item                      | Location                                                                                        |
| ------------------------- | ----------------------------------------------------------------------------------------------- |
| Test                      | `bgp/test_bgp_fact.py::test_bgp_facts`                                                          |
| Logic                     | `common/helpers/bgp.py::run_bgp_facts`                                                          |
| CLI options               | `conftest.py::pytest_addoption`                                                                 |
| `duthosts`                | `conftest.py::fixture_duthosts` (`name="duthosts"`, session)                                    |
| `duthost`                 | `conftest.py::duthost`                                                                          |
| `tbinfo` / `get_tbinfo`   | `conftest.py::tbinfo`, `get_tbinfo` -> `common/testbed.py::TestbedInfo`                         |
| `enhance_inventory`       | `conftest.py::enhance_inventory` (session, autouse)                                             |
| `ansible_adhoc`           | `common/plugins/ansible_fixtures.py::ansible_adhoc`                                             |
| `enum_*` fixtures         | `conftest.py::enum_frontend_dut_hostname`, `enum_asic_index` (just return `request.param`)      |
| Parametrization           | `conftest.py::pytest_generate_tests`                                                            |
| Plugins loaded            | `conftest.py::pytest_plugins` (sanity_check, loganalyzer, conditional_mark, custom_markers ...) |
| Topology/device_type skip | `common/plugins/custom_markers/__init__.py::check_topology`                                     |
| Host classes              | `common/devices/duthosts.py`, `multi_asic.py`, `sonic.py`, `sonic_asic.py`, `base.py`           |


### Example commands (from `sonic-mgmt/tests`)
For running i use my already running topology 

##### collect-only
command:
`./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c bgp/test_bgp_fact.py -f vtestbed.yaml -i ../ansible/veos_vtb -e "--collect-only"`
output: 
```
<Package tests>
  <Dir bgp>
    <Module test_bgp_fact.py>
      <Function test_bgp_facts[vlab-01-None]>
```

##### run test by filename
command:
`./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c bgp/test_bgp_fact.py -f vtestbed.yaml -i ../ansible/veos_vtb`

##### run test by ID
command:
`./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c bgp/test_bgp_fact.py::test_bgp_facts[vlab-01-None] -f vtestbed.yaml -i ../ansible/veos_vtb`

##### run test with debug mode
`-m debug`
command:
`./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c bgp/test_bgp_fact.py::test_bgp_facts[vlab-01-None] -f vtestbed.yaml -i ../ansible/veos_vtb -m debug`
output:
```
maksymperfeniuk@sonic_mgmt_container:/data/sonic-mgmt/tests$ ./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c bgp/test_bgp_fact.py::test_bgp_facts[vlab-01-None] -f vtestbed.yaml -i ../ansible/veos_vtb -m debug
pkill: pattern that searches for process name longer than 15 characters will result in zero matches
Try `pkill -f' option to match against the complete command line.
=== Clearing pytest cache ===
=== Show test settings ===
SCRIPT:                ./run_tests.sh
FULL_PATH:             /data/sonic-mgmt/tests/run_tests.sh
SCRIPT_PATH:           /data/sonic-mgmt/tests
BASE_PATH:             /data/sonic-mgmt
ANSIBLE_PARENT_DIR:    /data/sonic-mgmt
ANSIBLE_CONFIG:        /data/sonic-mgmt/ansible
ANSIBLE_LIBRARY:       /data/sonic-mgmt/ansible/library/
AUTO_RECOVER:          True
BSL:                   False
BYPASS_UTIL:           False
CLI_LOG_LEVEL:         warning
EXTRA_PARAMETERS:      
FILE_LOG_LEVEL:        debug
INCLUDE_FOLDERS:       
INVENTORY:             ../ansible/veos_vtb
LOG_PATH:              logs
OMIT_FILE_LOG:         False
RETAIN_SUCCESS_LOG:    False
SKIP_SCRIPTS:          
SKIP_FOLDERS:          ptftests acstests saitests scripts k8s sai_qualify
TEST_CASES:            bgp/test_bgp_fact.py::test_bgp_facts[vlab-01-None]
TEST_CASES_FILE:       
TEST_FILTER:           
TEST_INPUT_ORDER:      False
TEST_MAX_FAIL:         0
REPEAT_COUNT:          1
TEST_METHOD:           debug
TESTBED_FILE:          vtestbed.yaml
TEST_LOGGING_OPTIONS:  --junit-xml=logs/tr.xml --log-file=logs/test.log
TEST_TOPOLOGY_OPTIONS: 
PRET_LOGGING_OPTIONS:  --junit-xml=logs/pretest.xml --log-file=logs/pretest.log
POST_LOGGING_OPTIONS:  --junit-xml=logs/posttest.xml --log-file=logs/posttest.log
UTIL_TOPOLOGY_OPTIONS: --topology util
NO_CLEAR_CACHE:        False
MUX_COMBO_MODE:        False
PYTEST_COMMON_OPTS:    -c pytest.ini                       --inventory ../ansible/veos_vtb                       --host-pattern vlab-01                       --dpu-pattern None                       --testbed vms-kvm-t0                       --testbed_file vtestbed.yaml                       --log-cli-level warning                       --log-file-level debug                       --kube_master unset                       --showlocals                       --assert plain                       --show-capture no                       -rav --allow_recover --ignore=ptftests --ignore=acstests --ignore=saitests --ignore=scripts --ignore=k8s --ignore=sai_qualify
maksymperfeniuk@sonic_mgmt_container:/data/sonic-mgmt/tests$ 
```

