<p align="center">
<img src="https://i.imgur.com/pU5A58S.png" alt="Microsoft Active Directory Logo"/>
</p>

<h1>On-premises Active Directory Deployed in the Cloud (Azure)</h1>
This tutorial outlines the implementation of on-premises Active Directory within Azure Virtual Machines.<br />

<h2>Environments and Technologies Used</h2>

- Microsoft Azure (Virtual Machines/Compute)
- Remote Desktop
- Active Directory Domain Services
- PowerShell

<h2>Operating Systems Used </h2>

- Windows Server 2022
- Windows 10 (21H2)

<h2>High-Level Deployment and Configuration Steps</h2>

- Set up Domain Controller and Client
- Deploying Active Directory
- Creating a Domain Admin User
- Join the Client VM to the Domain
- Setup Remote Desktop for Non-Admin Users for Client
- Create Additional Users
- Dealing with Account Lockouts
- Enabling and Disabling Accounts
- Observing Logs

<h2>Deployment and Configuration Steps</h2>

![image](https://github.com/user-attachments/assets/a0d17964-e374-41a2-a743-46cea7fc8d5a)

<p>
First we're going to create a resource group through the Azure GUI. Just search for resource groups and create a new one. We're also going to create a VNet and Subnet. Again, just search for virtual networks and create a new one. From there, we can now create our VMs. One will act as the domain controller and the other as a client. Create the domain controller, making sure it's part of the resource group and VNet you created. It will run Windows Server 2022. Set whatever username and password you'd like, just remember what it is. Once it's created, we're going to set its NIC private IP address to static to make sure it doesn't get assigned a new one if it powers off and on. Click on the VM, go to network settings, click on the NIC, click on the IP address, and set the allocation to static. From there, use remote desktop to log into the VM so we can disable the firewall to test connectivity. Once logged in, search for wf.msc. Turn off the firewalls for all of the profiles and hit apply and okay. Now we'll create the client VM, which will run Windows 10 and use the same resource group and VNet as the controller. Be sure to go to the client's NIC and make sure that the DNS server isn't set to inherit from the VNet, but instead uses the private IP address of the domain controller. From here, restart the client VM on Azure. We'll now log into the client using remote desktop. Attempt to ping the domain controller's private IP address, and type in ipconfig /all to check to see if the domain controller is being used as the DNS server.
</p>
<br />

![image](https://github.com/user-attachments/assets/90924ae1-00b4-458a-86f5-ebcc2bb6ad75)

<p>
We're now going to install active directory. Log into your domain controller VM. From there, go to Server Manager and click on add roles and features, click on Active Directory Domain Services, check the restart if required, and install. We're now going to promote the VM to a domain controller. Within server manager, click on the flag in the upper right corner. Click promote this server and click add new forest. You can name it whatever you want but I went with mydomain.com. Set whatever DSRM password you'd like and install. Your VM should restart at this point and you should log in using the same username and password you set except with mydomain.com\ in front of the username as we are now logging into a domain user, so we need to specify the domain.
</p>
<br />

![image](https://github.com/user-attachments/assets/040eb5d6-0645-4a0d-9951-857f734ad4b1)

<p>
We're now going to create a domain admin user account, along with some organizational units. Within your domain controller VM, search for Active Directory Users and Computers, and create an OU for "_EMPLOYEES" and "_ADMINS" so we can organize our employees and admins. To do this, right click mydomain.com, and hit new and Organization Unit to create both of them. Within _ADMINS, create a new user called Jane Doe. The username we'll use is jane_admin and you can set whatever password you'd like. Deselect User must change password at next login and select Password never expires. This isn't good practice in general but it's fine for the sake of this tutorial to test things out. We're going to right click our Jane Doe user and select properties, then Member Of. Click add, type in domain admins and hit check names. From there, hit OK, Apply, and OK. Jane should now officially be a domain user and we will now be using this account for that purpose. Log out of your domain controller and log back in as mydomain.com\jane_admin.
</p>
<br />

![image](https://github.com/user-attachments/assets/a4cea430-e556-479b-8343-7afc815ea996)

<p>
We're now going to join our client VM to the domain. Log into the client as the original local admin you set. Right click on Start and go to System. Click on Rename This PC (Advanced), click Change and set the Member Of to Our Domain mydomain.com. You'll be prompted to log in as jane_admin to grant permission and your client will restart. Within your domain controller, go to ADUC and verify that your client is in the domain by clicking on mydomain.com and computers. Create an OU called "_CLIENTS" and drag your client into there.
</p>
<br />

![image](https://github.com/user-attachments/assets/b7adeab5-9856-41fd-a958-d431923971b6)

<p>
We're now going to allow non-admin users to remote desktop into the client. To do that, log into the client VM as mydomain.com\jane_admin. Go to system settings, remote desktop, and edit the users that can remotely access. Select add, type in domain users and check names.
</p>
<br />

![image](https://github.com/user-attachments/assets/2073fcf7-a8a8-4afc-98b5-fdcf22badf70)

<p>
We're now going to create a bunch of non-admin users and log into our client with one of them. Log into your domain controller using the mydomain.com\jane_admin account. Open up Powershell ISE as an admin and create and save a new file. Paste in the code from this link: https://github.com/joshmadakor1/AD_PS/blob/master/Generate-Names-Create-Users.ps1. Run the code and observe all the users being created. You should now see a lot of employees with the _EMPLOYEES OU within ADUC. You can control-c to stop the script if it's taking too long to create all of the users. From here, log into your client using one of the users that was just created with the mydomain.com\ preceding. The password will be Password1 for all of the accounts.
</p>
<br />

![image](https://github.com/user-attachments/assets/dee9da2b-6e5c-4d67-b160-581f18d9c8b4)

<p>
We're now going to demonstrate dealing with account lockouts. Log into the domain controller with your mydomain.com\jane_admin user. Pick a random user from the _EMPLOYEES OU. Try to log into that user's account using the wrong password with your client. Nothing should happen. To set a lockout policy, we're going to need to go into the domain controller and edit the policy that deals with that. Type in gpmc.msc into the start box of your domain controller and hit enter. This will open your group policy management. Right click the default domain policy and hit edit. Navigate to account lockout policy and change the account lockout duration to 30 minutes and hit apply. We now need to force the group policy to update on our client as opposed to waiting 90 minutes for it to happen automatically. Log into the client as mydomain.com\jane_admin and go to the command prompt. Type in gpudate /force and the group policy should update. If you try to log back in as a random user with the wrong password more than 5 times, the account will be locked. To unlock the account, go back to ADUC and find the user you just tried to log in with. Go to the account tab and select unlock this account. You should now be able to log in with the correct password.
</p>
<br />

![image](https://github.com/user-attachments/assets/2b59c9dd-8bfd-40cc-a5f5-2ea7099316c1)

<p>
To disable an account, right click the same random user you've been using from ADUC and click disable account. If you try to log in, it should fail and you'll get an error message. To reenable the account, just right click it again and select enable account. It's generally good practice to find out why an account was disabled before just reenabling it because there's usually a reason for it.
</p>
<br />

![image](https://github.com/user-attachments/assets/bfed0943-ea02-48bd-940a-4394989060d6)

<p>
We can also observe the logs of what we just did. In the domain controller, type eventvwr.msc in the start box, go to windows logs and security. Search within security for the random user you tried to log in with and observe the logs, like unlocking the account. This is a good way to see what's going on within the network.
</p>
<br />
