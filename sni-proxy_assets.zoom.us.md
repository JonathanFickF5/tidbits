# SNI Proxy for assets.zoom.us
This configuration is an example of a way to add SNI to a request from a client that does not support it.
Specifically, it's in support of @f5regan's excellent work at https://github.com/f5regan/o365-apm-split-tunnel

It's currently hard-coded specifically to Zoom, but it could be elegantly extended to support multiple external services/lists:
* Creating a new pool/monitor for each external list
* Using a BIG-IP LTM Policy to select the appropriate pool based on HTTP Host Header, and rewriting the Host Header to the public string
* Adding multiple server-ssl profiles, each associated with the single virtual server
 * BIG-IP 15.1+ will select the appropriate profile automatically based on SNI
 * Prior to 15.1, an iRule is required
This is untested and should be thoroughly tested in non-prod before use.


Modify as needed, and use `tmsh merge sys config from-terminal` or similar to load the following:

### Create an LTM monitor to make sure the resource availability
* Note the reference to the server ssl profile for SNI to function
* Look for a simple recieve string to mark up (any number, followed by /, then another number in the cidr mask)
* Be a good netizen and tell the remote server why we're monitoring them with a User-Agent
* In production the interval/timeout can probably be increased from seconds to minutes or even hours
```
ltm monitor https /Common/assets.zoom.us {
    adaptive disabled  
    defaults-from /Common/https
    destination *:* 
    interval 5               
    ip-dscp 0             
    recv [0-9]/[0-9][0-9]
    recv-disable none                
    send "GET /docs/ipranges/Zoom.txt HTTP/1.1\r\nHost:assets.zoom.us\r\nUser-Agent: F5 BIG-IP VPN Split Proxy\r\nConnection: Close\r\n\r\n"
    ssl-profile /Common/serverssl-assets.zoom.us
    time-until-up 0     
    timeout 16
}
```
### Create an FQDN pool for the server-side
```
ltm node /Common/assets.zoom.us {           
    fqdn {                  
        address-family all
        autopopulate enabled
        name assets.zoom.us
    }              
} 

ltm pool /Common/assets.zoom.us {           
    members {               
        /Common/assets.zoom.us:443 {
            fqdn {          
                autopopulate enabled
                name assets.zoom.us
            }   
        }             
    }                       
    monitor /Common/assets.zoom.us
}
```
### Create an HTTP profile to strip an existing Host header, and set our own
```
ltm profile http /Common/http-assets.zoom.us {
    app-service none   
    defaults-from /Common/http 
    header-erase Host
    header-insert "Host: assets.zoom.us"
    proxy-type reverse    
}
```
### Create a serverssl profile to force SNI to the server-side request
```
ltm profile server-ssl /Common/serverssl-assets.zoom.us {
    app-service none   
    defaults-from /Common/serverssl
    server-name assets.zoom.us
}
```

### Create a virtual server for the split proxy script to download from
This only needs to be accessible by the BIG-IPs with the split proxy script. This could be http or https.
```
ltm virtual /Common/sniproxy-assets.zoom.us {
    destination /Common/192.0.2.123:80
    ip-protocol tcp
    mask 255.255.255.255           
    pool /Common/assets.zoom.us                  
    profiles {        
        /Common/http-assets.zoom.us { }                  
        /Common/serverssl-assets.zoom.us {
            context serverside
        }
        /Common/tcp { }
    }
    source 0.0.0.0/0
    source-address-translation {
        type automap
    }
    translate-address enabled
    translate-port enabled
}
```
