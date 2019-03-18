# RHEL7-ControlGroup-Cgroup
RHEL 7 - Control Group - CGroup
Serviços systemd de RHEL7 possuem limitadores do uso de CPU em 3 formas:

	CPUQuota.....: % de CPU
	CPUShares....: equivalente a renice
	CPUAccounting: ???

De forma permanente: "isso altera os arquivos .conf"

	# systemctl set-property httpd.service CPUQuota=5%
	# systemctl set-property httpd.service CPUShares=500
	# systemctl set-property httpd.service CPUAccounting=yes
	
De forma temporária: "--runtime"

	# systemctl set-property --runtime httpd.service CPUQuota=5%
	# systemctl set-property --runtime httpd.service CPUShares=500
	# systemctl set-property --runtime httpd.service CPUAccounting=yes

Ele cria um arquivo novo no /etc/systemd/system/<SERVIÇO>.service.d/*

	# ls -l /etc/systemd/system/httpd.service.d/*
	-rw-r--r--. 1 root root 22 Mar 18 11:48 /etc/systemd/system/httpd.service.d/50-CPUQuota.conf
	-rw-r--r--. 1 root root 24 Mar 18 11:52 /etc/systemd/system/httpd.service.d/50-CPUShares.conf
	-rw-r--r--. 1 root root 28 Mar 18 11:53 /etc/systemd/system/httpd.service.d/50-CPUAccounting.conf

Listando cada conteúdo:

	# cat /etc/systemd/system/httpd.service.d/50-CPUAccounting.conf
	[Service]
	CPUAccounting=yes
	
	# cat /etc/systemd/system/httpd.service.d/50-CPUQuota.conf
	[Service]
	CPUQuota=5%
	
	# cat /etc/systemd/system/httpd.service.d/50-CPUShares.conf
	[Service]
	CPUShares=500

Para evidenciar cada item:

# systemd-cgtop
# top
# sar -P ALL 2

################################################
LAB1 by open package stress (não tem na RedHat):
################################################

# wget  http://download-ib01.fedoraproject.org/pub/fedora/linux/releases/29/Everything/x86_64/os/Packages/s/stress-1.0.4-21.fc29.x86_64.rpm
# yum localinstall -y stress-1.0.4-21.fc29.x86_64.rpm

### Creating user and group:

# useradd user1
# useradd user2
# useradd user3
# groupadd group1
# usermod -aG group1 user1
# usermod -aG group1 user2
# usermod -aG group1 user3

### Creating service file:
 
# cat > /etc/systemd/system/stress1.service << EEOOFF
> [Unit]
> Description=Estressando cpus e memórias
> After=network.target
>
> [Service]
> Type=forking
> User=user1
> Group=group1
> RemainAfterExit=no
> WorkingDirectory=/home/user1
> ExecStart=/bin/bash /home/user1/stress001.start.sh
> ExecStop=/usr/bin/pkill stress
>
> [Install]
> WantedBy=multi-user.target
> EEOOFF

### Creating Script file: 

# cat > /home/user1/stress1.start.sh << EEOOFF
> #!/bin/bash
> stress --vm 20 > /dev/null &
> echo 0
> EEOOFF

# chmod a+x /home/user1/stress1.start.sh

### Enabling and Starting stress1.service:

# systemctl enable stress1 --now

### Check CPU:

# sar -P ALL 5
# top

### Implementing CPUQuota (%) ONLINE without restart:

# systemctl set-property stress1 CPUQuota=33%

###  Check CPU:

# sar -P ALL 5
# top
# systemd-cgtop

### Notice: A new file created on /etc/systemd/system/stress1.service.d:

# cat /etc/systemd/system/stress1.service.d/50-CPUQuota.conf
[Service]
CPUQuota=33%

################################################
### LAB2 - Stress httpd:
################################################

# ip a # to capture IP
# ab -c 100 -n 1000000 http://192.168.152.129/index.html
# sar -P ALL 5
# top
# systemd-cgtop

O problema é que um único processo "ab" gargala em uma cpu.
Vamos encapsular o "ab" limitando por usuário - já que "ab" não é um systemd service.



