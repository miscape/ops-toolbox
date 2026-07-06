Get an instant Ubuntu VM with a single command. Multipass can launch and run virtual machines

To install Multipass, run the following command:
```bash
sudo snap install multipass
```

Run this command to create a vm:
```bash
multipass launch --cpus 2 --memory 2G --disk 20G -n my-vm
```
