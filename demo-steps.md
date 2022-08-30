# Demo steps


## Provision multiple VMs

Follow https://github.com/keylime/keylime-vagrant-ansible-tpm-emulator/blob/master/docs/vagrant-setup-guide-fedora.md

## Clone repos

Demo repo, on both VMs:

```shell
git clone https://github.com/mbestavros/intoto-demo.git demo
```

You'll need to update the SCP IPs in `owner_alice/create_layout.py` according to each machine. Use `hostname -I` to see the hostname for each machine.

Importer tool, on verifier VM:

```shell
git clone https://github.com/mbestavros/keylime-policy-importer.git
```

## Set up in-toto venv

On both machines:

```shell
cd demo
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Run the demo

Will take place across both VMs.

### Step 1: Agent machine

Compile layout

```shell
cd owner_alice
python create_layout.py
```

Create allowlist

```shell
cd ../functionary_bob
export IN_TOTO_LINK_CMD_EXEC_TIMEOUT='1000'
in-toto-run --step-name create_allowlist_on_agent --products /root/allowlist.txt --key bob -- /root/keylime-dev/scripts/create_allowlist.sh -o ~/allowlist.txt -h sha256sum
```

Let it run. Then, move to verifier:

```shell
in-toto-run --step-name move_allowlist_to_verifier --products /home/vagrant/allowlist.txt --key bob -- scp /root/allowlist.txt vagrant@192.168.122.<INSERT CORRECT IP HERE>:~
```

Trust the fingerprint and provide password `vagrant` when prompted.

### Step 2: Verifier machine

```shell
cd demo/functionary_carl
```

Move the transferred file to root:

```shell
in-toto-run --step-name move_allowlist_to_root --products /root/allowlist.txt --key carl -- cp /home/vagrant/allowlist.txt /root/
```

Create binary

```shell
in-toto-run --step-name create_binary --products /root/demo/test-binary.sh --key carl -- chmod +x /root/demo/test-binary.sh
```

Send binary to agent

```shell
in-toto-run --step-name move_binary_to_agent --products /home/vagrant/test-binary.sh --key carl -- scp /root/demo/test-binary.sh vagrant@192.168.122.<INSERT CORRECT IP HERE>:~
```

Update allowlist

```shell
in-toto-run --step-name update_allowlist --materials /root/demo/functionary_carl/* --products /root/keylime-policy-importer/keylime-policy.json --key carl -- python3 /root/keylime-policy-importer/importer.py -l /root/demo/functionary_carl/<find beginning hash>-move_binary_to_agent.link -a /root/allowlist.txt
```
