# AWS-Netbox

xample: https://medium.com/@stephenpaynter/netbox-webhooks-with-ansible-tower-eaa6dd98320

On Ansible:
- On the Template, set Variables "Prompt on launch" checkbox 
- User icon -> User Details -> Tokens -> Add -> Scope=Write
- Token: zdfy7plqUzJWdm58gOA1PCJJ9okKcn


On Netbox
- User icon -> Profile -> API Tokens
- User icon -> Admin -> Webhooks -> Add (DCIM->Interface, Enabled)



Netbox Data Model
##################

Interfaces:

data {
'id': 4333,
'url': '/api/dcim/interfaces/4333/', 
'device': OrderedDict([('id', 318), ('url', '/api/dcim/devices/318/'), ('name', 'ios-sw1'), ('display_name', 'ios-sw1')])
'name': 'Ethernet1/0',
'label': '',
'type': OrderedDict([('value', 'virtual'), ('label', 'Virtual')]),
'enabled': True,
'lag': None,
'mtu': None,
'mac_address': '',
'mgmt_only': False,
'description': 'Ethernet1/0 description',
'connected_endpoint_type': None,
'connected_endpoint': None,
'connection_status': None,
'cable': None,
'mode': OrderedDict([('value', 'access'), ('label', 'Access')]),
'untagged_vlan': OrderedDict([('id', 491), ('url', '/api/ipam/vlans/491/'), ('vid', 20), ('name', 'twenty'), ('display_name', 'twenty (20)')]),
'tagged_vlans': [],
'tags': [],
'count_ipaddresses': 0
}

{ 
 "extra_vars": 
  {
     "device": "{{ data['device']['name'] }}"
     "interface": "{{ data['name'] }}",
     "vlan_id": "{{ data['untagged_vlan']['vid'] }}",
     "vlan_name": "{{ data['untagged_vlan']['name'] }}",
     "event": "{{ event }}",
     "timestamp": "{{ timestamp }}",
     "data": "{{ data }}"
  },
  "limit": "{{ data['device']['name'] }}"
}

IP

data {
'id': 631,
'url': '/api/ipam/ip-addresses/631/',
'family': OrderedDict([('value', 4), ('label', 'IPv4')]),
'address': '12.1.1.1/24',
'vrf': None,
'tenant': None,
'status': OrderedDict([('value', 'active'), ('label', 'Active')]),
'role': None,
'assigned_object_type': 'dcim.interface',
'assigned_object_id': 4343,
'assigned_object': {'id': 4343, 'url': '/api/dcim/interfaces/4343/',
    'device': OrderedDict([('id', 320), ('url', '/api/dcim/devices/320/'), ('name', 'ios-rt2'), ('display_name', 'ios-rt2')]),
    'name': 'Ethernet0/1',
    'cable': None,
    'connection_status': None
    },
'nat_inside': None,
'nat_outside': None,
'dns_name': '',
'description': '12.1.1.1/24 description',
'tags': [],
'custom_fields': {},
'created': '2022-06-08',
'last_updated': '2022-06-08T14:04:45.065029Z'
}

{ 
 "extra_vars": 
  {
     "device": "{{ data['assigned_object']['device']['name'] }}",
     "interface": "{{ data['assigned_object']['name'] }}",
     "ip_address": "{{ data['address'] }}",
     "address_family": "{{ data['family']['value'] }}",
     "description": "{{ data['description'] }}",
     "event": "{{ event }}",
     "timestamp": "{{ timestamp }}",
     "data": "{{ data }}"
  },
  "limit": "{{ data['assigned_object']['device']['name'] }}"
}


SYSLOG-NG

ios-rt2:
Docs: 

logging buffered warnings
logging trap informational
logging origin-id hostname
logging host 172.16.1.60 vrf mgmt
# only needed to log every config command
archive
 log config
  logging enable
  notify syslog

# test
interface Ethernet0/2
 ip address 3.3.3.3 255.255.255.0
 no shutdown



syslog-ng:

ource s_net {
        udp( ip(0.0.0.0) port(514) );
};

destination d_netsyslog {
        file("/var/log/net-syslog" owner("root") group("adm") perm (0640));
};

destination d_controller32 {
  http(
        url("https://172.16.1.32/api/v2/job_templates/13/launch/")
        method("POST")
        tls(
          peer-verify(no)
        )
        user-agent("syslog-ng-trigger")
        headers("Content-Type: application/json", "Accept: */*", "Authorization: Bearer 0vN27zhftZUoERyQLK0eisUev2zK7G")
        body('{"extra_vars": {"foo": "bar"}, "limit": "${HOST}"}')
 );
};

destination d_controller34 {
  http(
        url("https://172.16.1.34/api/v2/job_templates/119/launch/")
        method("POST")
        tls(
          peer-verify(no)
        )
        user-agent("syslog-ng-trigger")
        headers("Content-Type: application/json", "Accept: */*", "Authorization: Bearer zdfy7plqUzJWdm58gOA1PCJJ9okKcn")
        body('{"extra_vars": {"device": "$HOST"}, "limit": "$HOST"}')
 );
};

filter cisco_rt1 {
        host("172.16.1.151") or host("ios-rt1");
};

filter cisco_rt2 {
        host("172.16.1.152") or host("ios-rt2");
};

filter msg_cisco_rt1 {
        match(".*from console by cisco.*" value("MESSAGE"))
};

filter msg_cisco_rt2 {
        match('%SYS-5-CONFIG_I: Configured from console by console' value("MESSAGE"))
};

log {
        source(s_net);
        filter(cisco_rt1);
        filter(msg_cisco_rt1);
        destination(d_controller32);
};

log {
        source(s_net);
        filter(cisco_rt2);
        filter(msg_cisco_rt2);
        destination(d_controller34);
};

log {
        source(s_net);
        destination(d_netsyslog);
};

