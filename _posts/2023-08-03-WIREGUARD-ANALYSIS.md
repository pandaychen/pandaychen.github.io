---
layout:     post
title:      Wireguard 实现原理与分析（一）
subtitle:   一个基于 golang 实现的 vpn-tunnel 分析
date:       2023-08-03
author:     pandaychen
catalog:    true
tags:
    - network
    - wireguard
---


##  Ox00    前言
WireGuard（简称 wg）是一种快速、现代、安全的 VPN 协议，基于 golang 的开源地址 [在此](https://git.zx2c4.com/wireguard-go)，本文探讨其 linux 下的配置和实现等细节

##  0x01   工作原理
WireGuard 以 UDP 实现，但是运行在 IP 层（即 ip-over-udp）。每个 Peer 都会生成一个 `wg0` 虚拟网卡，同时服务端会在物理网卡上监听 UDP `51820` 端口。应用程序的包发送到内核以后，如果地址是虚拟专用网内部的，那么就会交给 `wg0` 设备，WireGuard 就会把这个 IP 包封装成 WireGuard 的包，然后在 UDP 中发送出去，对方的 Peer 的内核收到这个 UDP 包后再反向操作，解包成为 IP 包，然后交给对应的应用程序。 WireGuard 实现的虚拟网卡就像 `eth0` 一样，可以使用标准的 Linux 工具操作，像是 `ip`, `ifconfig` 之类的命令。所以 WireGuard 也就不用实现 QoS 之类的功能，毕竟其他工具已经实现了

![ARCH](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/wireguard/how-wireguard-works.png)

从实现上看，wireguard 亦是先前介绍的基于 tun 虚拟网卡构建的 VPN 应用。再回想下前文介绍的 TUN 数据流程：

1.  首先，WireGuard 需要在系统中创建一块虚拟网卡，并配置好这个虚拟网卡的 IP 地址，掩码，网关不需要配置
2.  使用 WireGuard 连接另一台设备，两台互相 peer 对方并验证各自的公钥私钥是否正确，全部正确后成功建立 peer
3.  建立成功后，所有前往虚拟网卡的流量都将被重新封装后发往另一台设备，由另一台设备解封装然后得到数据报文并在内部查找路由并匹配报文目的地

上面 `3` 步为建立一个 WireGuard VPN 链接的过程，连接建立 ok 后，A 设备与 B 设备互相需要保证虚拟网卡的 IP 在相同网络位的地址段中，并且这个地址段被 WireGuard 的配置文件 `AllowedIPs` 所允许通过

如果你试图从 A 设备访问 B 设备的对端子网，你需要在 A 设备上配置系统路由，将系统三层网络的路由目的地指向对端虚拟 IP 地址，出接口为虚拟网卡，并且这个地址段必须被 WireGuard 的配置文件 `AllowedIPs` 所允许通过

####    基础结构
wireguard-go 的基础结构包括了公私钥、网络接口、UDP 连接、会话等。

-   公私钥：wireguard-go 使用 `Curve25519` 椭圆曲线加密算法生成公私钥对，用于加密和解密数据包
-   网络接口：wireguard-go 创建一个虚拟网络接口，该接口可以像物理网络接口一样接收和发送数据包
-   UDP 连接：wireguard-go 使用 UDP 协议进行数据包传输，它通过 UDP 连接实现数据包的发送和接收
-   会话：wireguard-go 使用会话来管理网络连接，会话包括了公私钥、网络接口、UDP 连接等信息

####    数据包处理
wireguard-go 的数据包处理包括了加密解密、封装解封装等过程。

-   加密解密：wireguard-go 使用 `ChaCha20` 加密算法和 `Poly1305` 消息认证码算法对数据包进行加密和解密。加密过程中，数据包首先被封装到一个新的数据包中，并加入了一些元数据，然后使用 `ChaCha20` 算法进行加密，最后使用 `Poly1305` 算法生成消息认证码。解密过程中，首先使用 `Poly1305` 算法验证消息认证码，然后使用 `ChaCha20` 算法进行解密，最后解封装数据包
-   封装解封装：wireguard-go 使用 WireGuard 协议对数据包进行封装和解封装。封装过程中，数据包被封装到一个新的数据包中，并加入了一些元数据，例如发送方公钥、接收方公钥、时间戳等。解封装过程中，首先根据接收方公钥和发送方公钥计算出会话密钥，然后使用会话密钥对数据包进行解密和解封装

####    网络连接管理
wireguard-go 的网络连接管理包括了会话管理、路由管理、握手协议等。

-   会话管理：wireguard-go 使用会话来管理网络连接，会话包括了公私钥、网络接口、UDP 连接等信息。会话可以通过公钥来唯一标识一个网络连接，可以添加、删除、更新等操作
-   路由管理：wireguard-go 使用内核路由表来管理网络路由，可以添加、删除、更新等操作。路由表中的路由信息包括了目标 IP 地址、子网掩码、网关等信息
-   握手协议：wireguard-go 使用 Noise 协议进行握手，该协议可以实现安全的密钥交换和会话建立。在握手过程中，发送方和接收方会交换公钥、时间戳、随机数等信息，然后根据这些信息计算出会话密钥


##  0x02    wireguard 组网
参考 [安装 Wireguard 并组建中心辐射型网络](https://naiv.fun/Ops/53.html)

####    安装
[Installation](https://www.wireguard.com/install/#centos-7-module-plus-module-kmod-module-dkms-tools)，此外，有开源项目已经提供了 wireguard 的一键 [配置脚本](https://github.com/angristan/wireguard-install)

```bash
#!/bin/bash

# Secure WireGuard server installer
# https://github.com/angristan/wireguard-install

RED='\033[0;31m'
ORANGE='\033[0;33m'
GREEN='\033[0;32m'
NC='\033[0m'

function isRoot() {
	if ["${EUID}" -ne 0 ]; then
		echo "You need to run this script as root"
		exit 1
	fi
}

function checkVirt() {
	if ["$(systemd-detect-virt)" == "openvz" ]; then
		echo "OpenVZ is not supported"
		exit 1
	fi

	if ["$(systemd-detect-virt)" == "lxc" ]; then
		echo "LXC is not supported (yet)."
		echo "WireGuard can technically run in an LXC container,"
		echo "but the kernel module has to be installed on the host,"
		echo "the container has to be run with some specific parameters"
		echo "and only the tools need to be installed in the container."
		exit 1
	fi
}

function checkOS() {
	source /etc/os-release
	OS="${ID}"
	if [[${OS} == "debian" || ${OS} == "raspbian" ]]; then
		if [[${VERSION_ID} -lt 10 ]]; then
			echo "Your version of Debian (${VERSION_ID}) is not supported. Please use Debian 10 Buster or later"
			exit 1
		fi
		OS=debian # overwrite if raspbian
	elif [[${OS} == "ubuntu" ]]; then
		RELEASE_YEAR=$(echo "${VERSION_ID}" | cut -d'.' -f1)
		if [[${RELEASE_YEAR} -lt 18 ]]; then
			echo "Your version of Ubuntu (${VERSION_ID}) is not supported. Please use Ubuntu 18.04 or later"
			exit 1
		fi
	elif [[${OS} == "fedora" ]]; then
		if [[${VERSION_ID} -lt 32 ]]; then
			echo "Your version of Fedora (${VERSION_ID}) is not supported. Please use Fedora 32 or later"
			exit 1
		fi
	elif [[${OS} == 'centos' ]] || [[ ${OS} == 'almalinux' ]] || [[ ${OS} == 'rocky' ]]; then
		if [[${VERSION_ID} == 7* ]]; then
			echo "Your version of CentOS (${VERSION_ID}) is not supported. Please use CentOS 8 or later"
			exit 1
		fi
	elif [[-e /etc/oracle-release]]; then
		source /etc/os-release
		OS=oracle
	elif [[-e /etc/arch-release]]; then
		OS=arch
	else
		echo "Looks like you aren't running this installer on a Debian, Ubuntu, Fedora, CentOS, AlmaLinux, Oracle or Arch Linux system"
		exit 1
	fi
}

function getHomeDirForClient() {
	local CLIENT_NAME=$1

	if [-z "${CLIENT_NAME}" ]; then
		echo "Error: getHomeDirForClient() requires a client name as argument"
		exit 1
	fi

	# Home directory of the user, where the client configuration will be written
	if [-e "/home/${CLIENT_NAME}" ]; then
		# if $1 is a user name
		HOME_DIR="/home/${CLIENT_NAME}"
	elif ["${SUDO_USER}" ]; then
		# if not, use SUDO_USER
		if ["${SUDO_USER}" == "root" ]; then
			# If running sudo as root
			HOME_DIR="/root"
		else
			HOME_DIR="/home/${SUDO_USER}"
		fi
	else
		# if not SUDO_USER, use /root
		HOME_DIR="/root"
	fi

	echo "$HOME_DIR"
}

function initialCheck() {
	isRoot
	checkVirt
	checkOS
}

function installQuestions() {
	echo "Welcome to the WireGuard installer!"
	echo "The git repository is available at: https://github.com/angristan/wireguard-install"
	echo ""
	echo "I need to ask you a few questions before starting the setup."
	echo "You can keep the default options and just press enter if you are ok with them."
	echo ""

	# Detect public IPv4 or IPv6 address and pre-fill for the user
	SERVER_PUB_IP=$(ip -4 addr | sed -ne 's|^.* inet \([^/]*\)/.* scope global.*$|\1|p' | awk '{print $1}' | head -1)
	if [[-z ${SERVER_PUB_IP} ]]; then
		# Detect public IPv6 address
		SERVER_PUB_IP=$(ip -6 addr | sed -ne 's|^.* inet6 \([^/]*\)/.* scope global.*$|\1|p' | head -1)
	fi
	read -rp "IPv4 or IPv6 public address:" -e -i "${SERVER_PUB_IP}" SERVER_PUB_IP

	# Detect public interface and pre-fill for the user
	SERVER_NIC="$(ip -4 route ls | grep default | grep -Po'(?<=dev)(\S+)'| head -1)"
	until [[${SERVER_PUB_NIC} =~ ^[a-zA-Z0-9_]+$ ]]; do
		read -rp "Public interface:" -e -i "${SERVER_NIC}" SERVER_PUB_NIC
	done

	until [[${SERVER_WG_NIC} =~ ^[a-zA-Z0-9_]+$ && ${#SERVER_WG_NIC} -lt 16 ]]; do
		read -rp "WireGuard interface name:" -e -i wg0 SERVER_WG_NIC
	done

	until [[${SERVER_WG_IPV4} =~ ^([0-9]{1,3}\.){3} ]]; do
		read -rp "Server WireGuard IPv4:" -e -i 10.66.66.1 SERVER_WG_IPV4
	done

	until [[${SERVER_WG_IPV6} =~ ^([a-f0-9]{1,4}:){3,4}: ]]; do
		read -rp "Server WireGuard IPv6:" -e -i fd42:42:42::1 SERVER_WG_IPV6
	done

	# Generate random number within private ports range
	RANDOM_PORT=$(shuf -i49152-65535 -n1)
	until [[${SERVER_PORT} =~ ^[0-9]+$ ]] && [ "${SERVER_PORT}" -ge 1 ] && [ "${SERVER_PORT}" -le 65535 ]; do
		read -rp "Server WireGuard port [1-65535]:" -e -i "${RANDOM_PORT}" SERVER_PORT
	done

	# Adguard DNS by default
	until [[${CLIENT_DNS_1} =~ ^((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$ ]]; do
		read -rp "First DNS resolver to use for the clients:" -e -i 1.1.1.1 CLIENT_DNS_1
	done
	until [[${CLIENT_DNS_2} =~ ^((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$ ]]; do
		read -rp "Second DNS resolver to use for the clients (optional):" -e -i 1.0.0.1 CLIENT_DNS_2
		if [[${CLIENT_DNS_2} == "" ]]; then
			CLIENT_DNS_2="${CLIENT_DNS_1}"
		fi
	done

	until [[${ALLOWED_IPS} =~ ^.+$ ]]; do
		echo -e "\nWireGuard uses a parameter called AllowedIPs to determine what is routed over the VPN."
		read -rp "Allowed IPs list for generated clients (leave default to route everything):" -e -i '0.0.0.0/0,::/0' ALLOWED_IPS
		if [[${ALLOWED_IPS} == "" ]]; then
			ALLOWED_IPS="0.0.0.0/0,::/0"
		fi
	done

	echo ""
	echo "Okay, that was all I needed. We are ready to setup your WireGuard server now."
	echo "You will be able to generate a client at the end of the installation."
	read -n1 -r -p "Press any key to continue..."
}

function installWireGuard() {
	# Run setup questions first
	installQuestions

	# Install WireGuard tools and module
	if [[${OS} == 'ubuntu' ]] || [[ ${OS} == 'debian' && ${VERSION_ID} -gt 10 ]]; then
		apt-get update
		apt-get install -y wireguard iptables resolvconf qrencode
	elif [[${OS} == 'debian' ]]; then
		if ! grep -rqs "^deb .* buster-backports" /etc/apt/; then
			echo "deb http://deb.debian.org/debian buster-backports main" >/etc/apt/sources.list.d/backports.list
			apt-get update
		fi
		apt update
		apt-get install -y iptables resolvconf qrencode
		apt-get install -y -t buster-backports wireguard
	elif [[${OS} == 'fedora' ]]; then
		if [[${VERSION_ID} -lt 32 ]]; then
			dnf install -y dnf-plugins-core
			dnf copr enable -y jdoss/wireguard
			dnf install -y wireguard-dkms
		fi
		dnf install -y wireguard-tools iptables qrencode
	elif [[${OS} == 'centos' ]] || [[ ${OS} == 'almalinux' ]] || [[ ${OS} == 'rocky' ]]; then
		if [[${VERSION_ID} == 8* ]]; then
			yum install -y epel-release elrepo-release
			yum install -y kmod-wireguard
			yum install -y qrencode # not available on release 9
		fi
		yum install -y wireguard-tools iptables
	elif [[${OS} == 'oracle' ]]; then
		dnf install -y oraclelinux-developer-release-el8
		dnf config-manager --disable -y ol8_developer
		dnf config-manager --enable -y ol8_developer_UEKR6
		dnf config-manager --save -y --setopt=ol8_developer_UEKR6.includepkgs='wireguard-tools*'
		dnf install -y wireguard-tools qrencode iptables
	elif [[${OS} == 'arch' ]]; then
		pacman -S --needed --noconfirm wireguard-tools qrencode
	fi

	# Make sure the directory exists (this does not seem the be the case on fedora)
	mkdir /etc/wireguard >/dev/null 2>&1

	chmod 600 -R /etc/wireguard/

	SERVER_PRIV_KEY=$(wg genkey)
	SERVER_PUB_KEY=$(echo "${SERVER_PRIV_KEY}" | wg pubkey)

	# Save WireGuard settings
	echo "SERVER_PUB_IP=${SERVER_PUB_IP}
SERVER_PUB_NIC=${SERVER_PUB_NIC}
SERVER_WG_NIC=${SERVER_WG_NIC}
SERVER_WG_IPV4=${SERVER_WG_IPV4}
SERVER_WG_IPV6=${SERVER_WG_IPV6}
SERVER_PORT=${SERVER_PORT}
SERVER_PRIV_KEY=${SERVER_PRIV_KEY}
SERVER_PUB_KEY=${SERVER_PUB_KEY}
CLIENT_DNS_1=${CLIENT_DNS_1}
CLIENT_DNS_2=${CLIENT_DNS_2}
ALLOWED_IPS=${ALLOWED_IPS}" >/etc/wireguard/params

	# Add server interface
	echo "[Interface]
Address = ${SERVER_WG_IPV4}/24,${SERVER_WG_IPV6}/64
ListenPort = ${SERVER_PORT}
PrivateKey = ${SERVER_PRIV_KEY}">"/etc/wireguard/${SERVER_WG_NIC}.conf"

	if pgrep firewalld; then
		FIREWALLD_IPV4_ADDRESS=$(echo "${SERVER_WG_IPV4}" | cut -d"." -f1-3)".0"
		FIREWALLD_IPV6_ADDRESS=$(echo "${SERVER_WG_IPV6}" | sed 's/:[^:]*$/:0/')
		echo "PostUp = firewall-cmd --add-port ${SERVER_PORT}/udp && firewall-cmd --add-rich-rule='rule family=ipv4 source address=${FIREWALLD_IPV4_ADDRESS}/24 masquerade'&& firewall-cmd --add-rich-rule='rule family=ipv6 source address=${FIREWALLD_IPV6_ADDRESS}/24 masquerade'
PostDown = firewall-cmd --remove-port ${SERVER_PORT}/udp && firewall-cmd --remove-rich-rule='rule family=ipv4 source address=${FIREWALLD_IPV4_ADDRESS}/24 masquerade' && firewall-cmd --remove-rich-rule='rule family=ipv6 source address=${FIREWALLD_IPV6_ADDRESS}/24 masquerade'" >>"/etc/wireguard/${SERVER_WG_NIC}.conf"
	else
		echo "PostUp = iptables -I INPUT -p udp --dport ${SERVER_PORT} -j ACCEPT
PostUp = iptables -I FORWARD -i ${SERVER_PUB_NIC} -o ${SERVER_WG_NIC} -j ACCEPT
PostUp = iptables -I FORWARD -i ${SERVER_WG_NIC} -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o ${SERVER_PUB_NIC} -j MASQUERADE
PostUp = ip6tables -I FORWARD -i ${SERVER_WG_NIC} -j ACCEPT
PostUp = ip6tables -t nat -A POSTROUTING -o ${SERVER_PUB_NIC} -j MASQUERADE
PostDown = iptables -D INPUT -p udp --dport ${SERVER_PORT} -j ACCEPT
PostDown = iptables -D FORWARD -i ${SERVER_PUB_NIC} -o ${SERVER_WG_NIC} -j ACCEPT
PostDown = iptables -D FORWARD -i ${SERVER_WG_NIC} -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o ${SERVER_PUB_NIC} -j MASQUERADE
PostDown = ip6tables -D FORWARD -i ${SERVER_WG_NIC} -j ACCEPT
PostDown = ip6tables -t nat -D POSTROUTING -o ${SERVER_PUB_NIC} -j MASQUERADE">>"/etc/wireguard/${SERVER_WG_NIC}.conf"
	fi

	# Enable routing on the server
	echo "net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1" >/etc/sysctl.d/wg.conf

	sysctl --system

	systemctl start "wg-quick@${SERVER_WG_NIC}"
	systemctl enable "wg-quick@${SERVER_WG_NIC}"

	newClient
	echo -e "${GREEN}If you want to add more clients, you simply need to run this script another time!${NC}"

	# Check if WireGuard is running
	systemctl is-active --quiet "wg-quick@${SERVER_WG_NIC}"
	WG_RUNNING=$?

	# WireGuard might not work if we updated the kernel. Tell the user to reboot
	if [[${WG_RUNNING} -ne 0 ]]; then
		echo -e "\n${RED}WARNING: WireGuard does not seem to be running.${NC}"
		echo -e "${ORANGE}You can check if WireGuard is running with: systemctl status wg-quick@${SERVER_WG_NIC}${NC}"
		echo -e "${ORANGE}If you get something like \"Cannot find device ${SERVER_WG_NIC}\", please reboot!${NC}"
	else # WireGuard is running
		echo -e "\n${GREEN}WireGuard is running.${NC}"
		echo -e "${GREEN}You can check the status of WireGuard with: systemctl status wg-quick@${SERVER_WG_NIC}\n\n${NC}"
		echo -e "${ORANGE}If you don't have internet connectivity from your client, try to reboot the server.${NC}"
	fi
}

function newClient() {
	# If SERVER_PUB_IP is IPv6, add brackets if missing
	if [[${SERVER_PUB_IP} =~ .*:.* ]]; then
		if [[${SERVER_PUB_IP} != *"["* ]] || [[ ${SERVER_PUB_IP} != *"]"* ]]; then
			SERVER_PUB_IP="[${SERVER_PUB_IP}]"
		fi
	fi
	ENDPOINT="${SERVER_PUB_IP}:${SERVER_PORT}"

	echo ""
	echo "Client configuration"
	echo ""
	echo "The client name must consist of alphanumeric character(s). It may also include underscores or dashes and can't exceed 15 chars."

	until [[${CLIENT_NAME} =~ ^[a-zA-Z0-9_-]+$ && ${CLIENT_EXISTS} == '0' && ${#CLIENT_NAME} -lt 16 ]]; do
		read -rp "Client name:" -e CLIENT_NAME
		CLIENT_EXISTS=$(grep -c -E "^### Client ${CLIENT_NAME}\$" "/etc/wireguard/${SERVER_WG_NIC}.conf")

		if [[${CLIENT_EXISTS} != 0 ]]; then
			echo ""
			echo -e "${ORANGE}A client with the specified name was already created, please choose another name.${NC}"
			echo ""
		fi
	done

	for DOT_IP in {2..254}; do
		DOT_EXISTS=$(grep -c "${SERVER_WG_IPV4::-1}${DOT_IP}" "/etc/wireguard/${SERVER_WG_NIC}.conf")
		if [[${DOT_EXISTS} == '0' ]]; then
			break
		fi
	done

	if [[${DOT_EXISTS} == '1' ]]; then
		echo ""
		echo "The subnet configured supports only 253 clients."
		exit 1
	fi

	BASE_IP=$(echo "$SERVER_WG_IPV4" | awk -F '.' '{ print $1"."$2"."$3}')
	until [[${IPV4_EXISTS} == '0' ]]; do
		read -rp "Client WireGuard IPv4: ${BASE_IP}." -e -i "${DOT_IP}" DOT_IP
		CLIENT_WG_IPV4="${BASE_IP}.${DOT_IP}"
		IPV4_EXISTS=$(grep -c "$CLIENT_WG_IPV4/32" "/etc/wireguard/${SERVER_WG_NIC}.conf")

		if [[${IPV4_EXISTS} != 0 ]]; then
			echo ""
			echo -e "${ORANGE}A client with the specified IPv4 was already created, please choose another IPv4.${NC}"
			echo ""
		fi
	done

	BASE_IP=$(echo "$SERVER_WG_IPV6" | awk -F '::' '{ print $1}')
	until [[${IPV6_EXISTS} == '0' ]]; do
		read -rp "Client WireGuard IPv6: ${BASE_IP}::" -e -i "${DOT_IP}" DOT_IP
		CLIENT_WG_IPV6="${BASE_IP}::${DOT_IP}"
		IPV6_EXISTS=$(grep -c "${CLIENT_WG_IPV6}/128" "/etc/wireguard/${SERVER_WG_NIC}.conf")

		if [[${IPV6_EXISTS} != 0 ]]; then
			echo ""
			echo -e "${ORANGE}A client with the specified IPv6 was already created, please choose another IPv6.${NC}"
			echo ""
		fi
	done

	# Generate key pair for the client
	CLIENT_PRIV_KEY=$(wg genkey)
	CLIENT_PUB_KEY=$(echo "${CLIENT_PRIV_KEY}" | wg pubkey)
	CLIENT_PRE_SHARED_KEY=$(wg genpsk)

	HOME_DIR=$(getHomeDirForClient "${CLIENT_NAME}")

	# Create client file and add the server as a peer
	echo "[Interface]
PrivateKey = ${CLIENT_PRIV_KEY}
Address = ${CLIENT_WG_IPV4}/32,${CLIENT_WG_IPV6}/128
DNS = ${CLIENT_DNS_1},${CLIENT_DNS_2}

[Peer]
PublicKey = ${SERVER_PUB_KEY}
PresharedKey = ${CLIENT_PRE_SHARED_KEY}
Endpoint = ${ENDPOINT}
AllowedIPs = ${ALLOWED_IPS}">"${HOME_DIR}/${SERVER_WG_NIC}-client-${CLIENT_NAME}.conf"

	# Add the client as a peer to the server
	echo -e "\n### Client ${CLIENT_NAME}
[Peer]
PublicKey = ${CLIENT_PUB_KEY}
PresharedKey = ${CLIENT_PRE_SHARED_KEY}
AllowedIPs = ${CLIENT_WG_IPV4}/32,${CLIENT_WG_IPV6}/128">>"/etc/wireguard/${SERVER_WG_NIC}.conf"

	wg syncconf "${SERVER_WG_NIC}" <(wg-quick strip "${SERVER_WG_NIC}")

	# Generate QR code if qrencode is installed
	if command -v qrencode &>/dev/null; then
		echo -e "${GREEN}\nHere is your client config file as a QR Code:\n${NC}"
		qrencode -t ansiutf8 -l L <"${HOME_DIR}/${SERVER_WG_NIC}-client-${CLIENT_NAME}.conf"
		echo ""
	fi

	echo -e "${GREEN}Your client config file is in ${HOME_DIR}/${SERVER_WG_NIC}-client-${CLIENT_NAME}.conf${NC}"
}

function listClients() {
	NUMBER_OF_CLIENTS=$(grep -c -E "^### Client" "/etc/wireguard/${SERVER_WG_NIC}.conf")
	if [[${NUMBER_OF_CLIENTS} -eq 0 ]]; then
		echo ""
		echo "You have no existing clients!"
		exit 1
	fi

	grep -E "^### Client" "/etc/wireguard/${SERVER_WG_NIC}.conf" | cut -d ''-f 3 | nl -s') '
}

function revokeClient() {
	NUMBER_OF_CLIENTS=$(grep -c -E "^### Client" "/etc/wireguard/${SERVER_WG_NIC}.conf")
	if [[${NUMBER_OF_CLIENTS} == '0' ]]; then
		echo ""
		echo "You have no existing clients!"
		exit 1
	fi

	echo ""
	echo "Select the existing client you want to revoke"
	grep -E "^### Client" "/etc/wireguard/${SERVER_WG_NIC}.conf" | cut -d ''-f 3 | nl -s') '
	until [[${CLIENT_NUMBER} -ge 1 && ${CLIENT_NUMBER} -le ${NUMBER_OF_CLIENTS} ]]; do
		if [[${CLIENT_NUMBER} == '1' ]]; then
			read -rp "Select one client [1]:" CLIENT_NUMBER
		else
			read -rp "Select one client [1-${NUMBER_OF_CLIENTS}]:" CLIENT_NUMBER
		fi
	done

	# match the selected number to a client name
	CLIENT_NAME=$(grep -E "^### Client" "/etc/wireguard/${SERVER_WG_NIC}.conf" | cut -d ''-f 3 | sed -n"${CLIENT_NUMBER}"p)

	# remove [Peer] block matching $CLIENT_NAME
	sed -i "/^### Client ${CLIENT_NAME}\$/,/^$/d" "/etc/wireguard/${SERVER_WG_NIC}.conf"

	# remove generated client file
	HOME_DIR=$(getHomeDirForClient "${CLIENT_NAME}")
	rm -f "${HOME_DIR}/${SERVER_WG_NIC}-client-${CLIENT_NAME}.conf"

	# restart wireguard to apply changes
	wg syncconf "${SERVER_WG_NIC}" <(wg-quick strip "${SERVER_WG_NIC}")
}

function uninstallWg() {
	echo ""
	echo -e "\n${RED}WARNING: This will uninstall WireGuard and remove all the configuration files!${NC}"
	echo -e "${ORANGE}Please backup the /etc/wireguard directory if you want to keep your configuration files.\n${NC}"
	read -rp "Do you really want to remove WireGuard? [y/n]:" -e REMOVE
	REMOVE=${REMOVE:-n}
	if [[$REMOVE == 'y']]; then
		checkOS

		systemctl stop "wg-quick@${SERVER_WG_NIC}"
		systemctl disable "wg-quick@${SERVER_WG_NIC}"

		if [[${OS} == 'ubuntu' ]]; then
			apt-get remove -y wireguard wireguard-tools qrencode
		elif [[${OS} == 'debian' ]]; then
			apt-get remove -y wireguard wireguard-tools qrencode
		elif [[${OS} == 'fedora' ]]; then
			dnf remove -y --noautoremove wireguard-tools qrencode
			if [[${VERSION_ID} -lt 32 ]]; then
				dnf remove -y --noautoremove wireguard-dkms
				dnf copr disable -y jdoss/wireguard
			fi
		elif [[${OS} == 'centos' ]] || [[ ${OS} == 'almalinux' ]] || [[ ${OS} == 'rocky' ]]; then
			yum remove -y --noautoremove wireguard-tools
			if [[${VERSION_ID} == 8* ]]; then
				yum remove --noautoremove kmod-wireguard qrencode
			fi
		elif [[${OS} == 'oracle' ]]; then
			yum remove --noautoremove wireguard-tools qrencode
		elif [[${OS} == 'arch' ]]; then
			pacman -Rs --noconfirm wireguard-tools qrencode
		fi

		rm -rf /etc/wireguard
		rm -f /etc/sysctl.d/wg.conf

		# Reload sysctl
		sysctl --system

		# Check if WireGuard is running
		systemctl is-active --quiet "wg-quick@${SERVER_WG_NIC}"
		WG_RUNNING=$?

		if [[${WG_RUNNING} -eq 0 ]]; then
			echo "WireGuard failed to uninstall properly."
			exit 1
		else
			echo "WireGuard uninstalled successfully."
			exit 0
		fi
	else
		echo ""
		echo "Removal aborted!"
	fi
}

function manageMenu() {
	echo "Welcome to WireGuard-install!"
	echo "The git repository is available at: https://github.com/angristan/wireguard-install"
	echo ""
	echo "It looks like WireGuard is already installed."
	echo ""
	echo "What do you want to do?"
	echo "1) Add a new user"
	echo "2) List all users"
	echo "3) Revoke existing user"
	echo "4) Uninstall WireGuard"
	echo "5) Exit"
	until [[${MENU_OPTION} =~ ^[1-5]$ ]]; do
		read -rp "Select an option [1-5]:" MENU_OPTION
	done
	case "${MENU_OPTION}" in
	1)
		newClient
		;;
	2)
		listClients
		;;
	3)
		revokeClient
		;;
	4)
		uninstallWg
		;;
	5)
		exit 0
		;;
	esac
}

# Check for root, virt, OS...
initialCheck

# Check if WireGuard is already installed and load params
if [[-e /etc/wireguard/params]]; then
	source /etc/wireguard/params
	manageMenu
else
	installWireGuard
fi
```

按照上面的脚本进行部署，以 centos 三台机器组网（其中 `1` 台在公网，`2` 台非同一区域内网机器之间构建局域网通信）为例，拆分一下具体配置的步骤如下，假设公网的网络接口为 `eth0`，公共 IP 为 `1.1.1.1`，VPN 网段默认为 `10.66.66.0/24`，wg 服务端的配置文件在 `/etc/wireguard/wg0.conf`，上述一键脚本会生成 `wg0-client-username.conf` 文件，这个先 mark

wg 客户端，可以使用上面的 `wg0-client-username.conf` 模版文件，通常配置在 `/etc/wireguard/wg0.conf` 下

1、服务端配置（公网）：PeerA

```bash
[Interface]
Address = 10.66.66.1/24,fd42:42:42::1/64
ListenPort = 10086
PrivateKey = PeerA 私钥，脚本会自动生成
PostUp = iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT; iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE   #启动 wg0，由脚本自动生成
PostDown = iptables -D FORWARD -i eth0 -o wg0 -j ACCEPT; iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE    #关闭 wg0 时，由脚本自动生成

### Client PeerB
[Peer]
PublicKey = PeerB 公钥，脚本自动生成
PresharedKey = PeerB 预共享密钥，脚本自动生成
AllowedIPs = 10.66.66.2/32,fd42:42:42::2/128

### Client PeerC
[Peer]
PublicKey =PeerC 公钥，脚本自动生成
PresharedKey = PeerC 预共享密钥，脚本自动生成
AllowedIPs = 10.66.66.3/32,fd42:42:42::3/128
```


2、内网机器 B：PeerB

```BASH
[Interface]
PrivateKey = PeerB 私钥，脚本自动在公网端生成
Address = 10.66.66.2/32,fd42:42:42::2/128
DNS = 114.114.114.114,94.140.15.15

[Peer]
PublicKey = PeerA 公钥，脚本自动生成
PresharedKey = 预共享密钥，脚本自动生成
Endpoint = Peer 公网 IP 或域名: 10086
AllowedIPs = 0.0.0.0/0,::/0
```

3、内网机器 C：PeerC

```BASH
[Interface]
PrivateKey = PeerC 公钥，脚本自动在公网端生成
Address = 10.66.66.3/32,fd42:42:42::2/128
DNS = 114.114.114.114,94.140.15.15

[Peer]
PublicKey = PeerA 私钥，脚本自动生成
PresharedKey = 预共享密钥，脚本自动生成
Endpoint = Peer 公网 IP 或域名: 10086
AllowedIPs = 0.0.0.0/0,::/0
```


WireGuard 的实现中还有一个比较重要的配置叫做 `AllowedIPs` 是一个 IP 地址列表，表示允许哪些 IP 地址的流量通过 WireGuard 虚拟网络

####    wireguard 路由配置分析


1、服务端路由

服务端的启动日志如下：
```bash
ip link add wg0 type wireguard   # 创建一个 wireguard 设备
wg setconf wg0 /dev/fd/63        # 设置 wireguard 设备的配置
ip -4 address add 10.66.66.1 dev wg0   # 为 wireguard 设备添加一个 ip 地址
ip link set mtu 1420 up dev wg0        # 设置 wireguard 设备的 mtu
ip -4 route add 10.66.66.2/32 dev wg0  # 为 wireguard peer1 添加路由
ip -4 route add 10.66.66.3/32 dev wg0  # 为 wireguard peer2 添加路由
# 下面这几条 iptables 命令为 wireguard 设备添加 NAT 规则，使其成为 WireGuard 虚拟网络的默认网关
# 并使虚拟网络内的其他 peers 能通过此默认网关访问外部网络。
[#] iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

上面服务端初始化的日志操作如下：

1.  创建了 WireGuard 设备 `wg0` 并绑定了地址 `10.13.13.1`。作为 WireGuard 网络中的服务端，它所创建的这个 `wg0` 的任务是成为整个 WireGuard 虚拟网络的默认网关，处理来自虚拟网络内的其他 peers 的流量，构成一个星型网络
2.  服务端为它所关联的 peer1/peer2 添加了一个路由，使得 peer1/peer2 的流量能够被正确路由到 `wg0` 上
3.  为了让 WireGuard 虚拟网络内的其他 peers 的流量能够通过 `wg0` 设备访问外部网络或者互相访问，服务端为 `wg0` 添加了如下的 iptables 规则（这段很有用）
    -   `iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT`：允许进出 `wg0` 设备的数据包通过 netfilter 的 `FORWARD` 链（默认规则是 `DROP`，即默认是不允许通过的）
    -   `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`：在 `eth0` 网卡上添加 `MASQUERADE` 规则，即将数据包的源地址伪装成 `eth0` 网卡的地址，目的是为了允许 wireguard 的数据包通过 NAT 访问外部网络；而返回（响应）的流量会被 NAT 的 `conntrack` 链接追踪规则自动允许通过，不过 `conntrack` 表有自动清理机制，长时间没流量的话会被从 `conntrack` 表中移除


2、客户端路由



####    多网卡配置方式

![multi-nic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/wireguard/how-wireguard-works-1.png)

####    路由方式
[WireGuard 基本原理](https://cshihong.github.io/2020/10/11/WireGuard%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/)


##  0x02    代码
wireguard-go 的核心协议栈也是基于 gvisor[实现的](https://github.com/WireGuard/wireguard-go/blob/master/go.mod#L10)

-   tun 相关接口代码：[tun.go](https://git.zx2c4.com/wireguard-go/tree/tun/netstack/tun.go)


##  0x0 值得借鉴的地方

####    wireguard 的路由配置方式

####	透明代理中的DNS


##  0x0 参考
-   [wireguard 基本原理和最佳实践](https://www.ctyun.cn/developer/article/400185847689285)
-   [Routing & Network Namespace Integration](https://www.wireguard.com/netns/)
-   [WireGuard is not only for VPN](https://amod-kadam.medium.com/wireguard-is-not-only-for-vpn-9394605f458f)
-   [Quick Start](https://www.wireguard.com/quickstart/)
-   [WireGuard 教程：使用 DNS-SD 进行 NAT-to-NAT 穿透](https://icloudnative.io/posts/wireguard-endpoint-discovery-nat-traversal/)
-   [为什么 wireguard-go 高尚而 boringtun 孬种](https://zhuanlan.zhihu.com/p/548053546)
-   [WireGuard 教程：WireGuard 的工作原理](https://icloudnative.io/posts/wireguard-docs-theory/)
-   [Nebula: Open Source Overlay Networking](https://nebula.defined.net/docs/)
-   [WireGuard 到底好在哪？](https://zhuanlan.zhihu.com/p/404402933)
-   [吞吐优化札记](https://zhuanlan.zhihu.com/p/515881346)
-   [WireGuard VPN tunnel between 2 network islands](https://www.wut.de/e-55www-27-apus-000.php)
-   [WireGuard 教程：使用 DNS-SD 进行 NAT-to-NAT 穿透](https://icloudnative.io/posts/wireguard-endpoint-discovery-nat-traversal/)
-   [WireGuard 基本原理](https://cshihong.github.io/2020/10/11/WireGuard%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/)
-   [WIREGUARD SITE TO SITE CONFIGURATION](https://www.procustodibus.com/blog/2020/12/wireguard-site-to-site-config/)
-   [MULTI-HOP WIREGUARD](https://www.procustodibus.com/blog/2022/06/multi-hop-wireguard/)
-   [安装 Wireguard 并组建中心辐射型网络](https://naiv.fun/Ops/53.html)、
-   [WireGuard 教程：使用 Netmaker 来管理 WireGuard 的配置](https://icloudnative.io/posts/configure-a-mesh-network-with-netmaker/)
-   [Linux 上的 WireGuard 网络分析（一）](https://thiscute.world/posts/wireguard-on-linux/)
-	[WireGuard 教程：WireGuard 的工作原理](https://icloudnative.io/posts/wireguard-docs-theory/)
-	[WireGuard基本原理](https://cshihong.github.io/2020/10/11/WireGuard%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/)
-	[Clash DNS 科普](https://github.com/mritd/tpclash/wiki/Clash-DNS-%E7%A7%91%E6%99%AE)
-	[【WireGuard 白皮书带读 1】摘要 第一章](https://zhuanlan.zhihu.com/p/472525876)
-	[WireGuard 基础教程：wg-quick 路由策略解读](https://icloudnative.io/posts/linux-routing-of-wireguard/)