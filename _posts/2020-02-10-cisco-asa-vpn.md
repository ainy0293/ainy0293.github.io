---
layout: post
title: "ASA 8.4 版本 EZ VPN 配置"
subtitle: 'cisco asa vpn 配置，基于ipsec/l2tp，用于remote access远程接入vpn，实现在任何地方接入内网。'
author: "Ainy"
header-style: text
tags:
  - vpn
  - cisco
---

### 配置 crypto 策略

###### 第一阶段协商，或是做ipsec，需保证两端一致，否则协商不会成功

```
crypto ikev1 policy 10
  authentication pre-share
  encryption 3des
  hash md5
  group 2
  lifetime 86400
```
###### 配置 crypto 动态map和 map

```
crypto ipsec ikev1 transform-set ezvpn_set esp-3des esp-md5-hmac
crypto dynamic-map ezvpn_dymap 10 set ikev1 transform-set ezvpn_set
crypto map ezvpn_map 10 ipsec-isakmp dynamic ezvpn_dymap
```
###### 将 ikev1 和 crypto map 应用到出接口
```
crypto ikev1 enable outside
crypto map ezvpn_map interface outside
```

###### 配置 tunnel group 及 pre-shared 密钥

```
tunnel-group ezvpn_group type remote-access
tunnel-group ezvpn_group ipsec-attributes
  ikev1 pre-shared-key qq123456
```



### 用户相关

###### 允许用户访问的内网网段
```
access-list Split_tunnel_list extended permit ip 10.22.13.0 255.255.255.0 any
access-list Split_tunnel_list extended permit ip 10.22.15.0 255.255.255.0 any
access-list Split_tunnel_list extended permit host 10.27.128.239 any
```

###### 给予用户分配的IP池

```
ip local pool net_vpn_1 10.255.254.11-10.255.254.126 mask 255.255.255.128

```

###### 策略配置
```
group-policy ezvpn_group_policy_user1 internal
group-policy ezvpn_group_policy_user1 attributes
  address-pools value net_vpn_1
  split-tunnel-policy tunnelspecified
  split-tunnel-network-list value Split_tunnel_list
```

###### 配置用户并且关联到策略

```
username user1 password cisco123
username user1 attributes
  vpn-group-policy ezvpn_group_policy_user1
```

### NAT 免除 no nat

将所有内网会用到的网段关，和所有根据不同策略的用户的地址池的地址，来做 object

多个网段可以使用 object-group  单个网段可直接用 object network 即可

```
object-group network nonat_source
  network-object 10.22.13.0 255.255.255.0
  network-object 10.22.15.0 255.255.255.0
  network-object 10.27.128.224 255.255.255.240
  network-object 10.39.176.32 255.255.255.224
  exit

object-group network nonat_dest
  network-object 10.255.254.0 255.255.255.224
  network-object 10.255.253.0 255.255.255.0
  exit

nat (inside,outside) source static nonat_source nonat_source destination static nonat_dest nonat_dest no-proxy-arp

```





## 扩展配置：  针对不同用户设置不同的内网访问权限

> 其实也就是通过access-list 访问控制列表来实现，也可叫做感兴趣流

###### 需求描述：公司内网划分共有4个网段，其列表和用途如下：

1. 10.21.18.0/25      #测试A组使用
2. 10.21.18.128/25  #测试B组使用
3. 10.21.19.0/24      # 研发部门使用
4. 10.21.20.0/24  #生产服务器使用

###### 需求：
1. 测试A组和B组分别分配3个账号，他们只可以访问自己网段的
2. 研发部门12个账号，只允许访问研发部门的网段
3. 运维部门 4 个账号，可以访问所有网段
4. 两台公共服务器，OA 和 文件服务器，所有账号都可以访问
5. OA的 ip 为 10.21.20.16， 文件服务器为 10.21.20.135



##### 实现思路：针对不需求做不同的访问控制列表，创建 group-policy，并绑定到不同的用户



###### 配置地址池

> 配置不同用户的分配的地址池，也可以只用一个地址池，具体看需求，这里分多个，以部门来分

```
ip local pool testA_ipools 10.255.254.1-10.255.254.6 mask 255.255.255.248
ip local pool testB_ipools 10.255.254.9-10.255.254.14 mask 255.255.2555.248
ip local pool developer_ipools 10.255.254.17-10.255.254.30 mask 255.255.255.240
ip local pool yunwei_ipools 10.255.255.254.33-10.255.254.38 mask 255.255.255.248
```


###### 配置访问控制列表

> 根据部门来做访问控制列表，源地址为允许访问的服务器地址或网段，目标可以写上面的地址池网段，也可以写 any

```
access-list testA_access extended permit ip 10.21.18.0 255.255.255.128 any
access-list testA_access extended permit ip host 10.21.20.16 any
access-list testA_access extended permit ip host 10.21.20.135 any

access-list testB_access extended permit ip 10.21.18.128 255.255.255.128 any
access-list testB_access extended permit ip host 10.21.20.16 any
access-list testB_access extended permit ip host 10.21.20.135 any

access-list developer_access extended permit ip 10.21.19.0 255.255.255.0  any
access-list developer_access extended permit ip host 10.21.20.16 any
access-list developer_access extended permit ip host 10.21.20.135 any

access-list yunwei_access extended permit ip 10.21.18.0 255.255.254.0 any
access-list yunwei_access extended permit ip 10.21.20.0 255.255.255.0 any

```



###### 配置 测试 A 组的 group policy

```
group-policy testA_group_policy internal
group-policy testA_group_policy attributes
  address-pools value testA_ipools
  split-tunnel-policy tunnelspecified
  split-tunnel-network-list value testA_access
```



###### 配置 测试 B 组的 group policy

```
group-policy testB_group_policy internal
group-policy testB_group_policy attributes
  address-pools value testB_ipools
  split-tunnel-policy tunnelspecified
  split-tunnel-network-list value testB_access
```



###### 配置 研发 组的 group policy

```
group-policy developer_group_policy internal
group-policy developer_group_policy attributes
  address-pools value developer_ipools
  split-tunnel-policy tunnelspecified
  split-tunnel-network-list value developer_access
```



###### 配置 运维组的 group policy

```
group-policy yunwei_group_policy internal
group-policy yunwei_group_policy attributes
  address-pools value yunwei_ipools
  split-tunnel-policy tunnelspecified
  split-tunnel-network-list value yunwei_access
```


##### 创建用户并分配策略

###### 测试A组用户

```
username tom password cisco123
username tom attributes
  pn-group-policy testA_group_policy
  exit

username jack password cisco456
username jack 
  testA_group_policy testA_group_policy

```



###### 测试B组用户

```
username rose password cisco123
username rose attributes
  pn-group-policy testB_group_policy
  exit

username tony password cisco456
username tony 
  testA_group_policy testB_group_policy

```
> ................................
> 其它用户照上面例子配置即可，
> ................................

