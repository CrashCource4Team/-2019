# Creating an image of a virtual machine using Azure CLI

# Deprovision the VM

Connect to your Linux VM with an SSH client.

In the SSH window, enter the following command:
```sudo waagent -deprovision+user```

The +user parameter also removes the last provisioned user account. To keep user account credentials in the VM, use only -deprovision.

Enter y to continue. You can add the -force parameter to avoid this confirmation step.

After the command completes, enter exit to close the SSH client. The VM will still be running at this point.

# Create VM image

Deallocate the VM:

```
az vm deallocate \
  --resource-group myResourceGroup \
  --name myVM
  ```
  Wait for the VM to completely deallocate.

  Mark the VM as generalized:

```
az vm generalize \
  --resource-group myResourceGroup \
  --name myVM
  ```
  A VM that has been generalized can no longer be restarted.

  Create an image of the VM:

  ```
  az image create \
  --resource-group myResourceGroup \
  --name myImage --source myVM
```