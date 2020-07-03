---
title: 'Istio sidecar iptables and traffic steering detail'
subtitle: 'Tranlated by google'
summary: How istio manipulates the traffic.
authors:
- admin
tags:
- istio
- envoy
categories: []
date: "2020-06-25T00:00:00Z"
lastmod: "2020-06-25T00:00:00Z"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal point options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
image:
  caption: 'My tech blog https://cnadn.net'
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []

# Set captions for image gallery, use {{< gallery >}} to refrence in body.
gallery_item:
- album: gallery
  caption: Default
  image: theme-default.png
- album: gallery
  caption: Ocean
  image: theme-ocean.png
- album: gallery
  caption: Forest
  image: theme-forest.png
- album: gallery
  caption: Dark
  image: theme-dark.png
- album: gallery
  caption: Apogee
  image: theme-apogee.png
- album: gallery
  caption: 1950s
  image: theme-1950s.png
- album: gallery
  caption: Coffee theme with Playfair font
  image: theme-coffee-playfair.png
- album: gallery
  caption: Strawberry
  image: theme-strawberry.png
---

## Istio injected iptables
Istio implements the hijacking and processing of traffic by injecting the init container and envoy proxy container into the business pod. After the init container runs, the following NAT table rules for iptables will be generated in the corresponding linux namespace

```
&#91;root@k8s-node1-v1-16 ~]# iptables -t nat -L -v
Chain PREROUTING (policy ACCEPT 192K packets, 12M bytes)
 pkts bytes target     prot opt in     out     source               destination
 192K   12M ISTIO_INBOUND  tcp  --  any    any     anywhere             anywhere

Chain INPUT (policy ACCEPT 192K packets, 12M bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 40673 packets, 3694K bytes)
 pkts bytes target     prot opt in     out     source               destination
 8917  535K ISTIO_OUTPUT  tcp  --  any    any     anywhere             anywhere

Chain POSTROUTING (policy ACCEPT 40673 packets, 3694K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:ssh
  356 21360 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15090
 192K   11M RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15021
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15020
   34  2040 ISTIO_IN_REDIRECT  tcp  --  any    any     anywhere             anywhere

Chain ISTIO_IN_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
   34  2040 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15006

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  any    lo      127.0.0.6            anywhere    
    0     0 ISTIO_IN_REDIRECT  all  --  any    lo      anywhere            !localhost            owner UID match 1337
    0     0 RETURN     all  --  any    lo      anywhere             anywhere             ! owner UID match 1337
 8917  535K RETURN     all  --  any    any     anywhere             anywhere             owner UID match 1337
    0     0 ISTIO_IN_REDIRECT  all  --  any    lo      anywhere            !localhost            owner GID match 1337
    0     0 RETURN     all  --  any    lo      anywhere             anywhere             ! owner GID match 1337
    0     0 RETURN     all  --  any    any     anywhere             anywhere             owner GID match 1337
    0     0 RETURN     all  --  any    any     anywhere             localhost
    0     0 ISTIO_REDIRECT  all  --  any    any     anywhere             anywhere

Chain ISTIO_REDIRECT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001
```

## Outbound flow control
When the business container sends the request to the outside, such as productpage to reviews: 9080 port access, this connection will be redirected by iptables to port 127.0.0.1:115001, and then processed by envoy.

> REDIRECT
> This target is only valid in the nat table, in the PREROUTING and OUTPUT chains, and user-defined chains which are only called from those chains. It redirects the packet to the machine itself by changing the destination IP to the primary address of the incoming interface (locally-generated packets are mapped to the localhost address, 127.0.0.1 for IPv4 and ::1 for IPv6).
> 
> https://ipset.netfilter.org/iptables-extensions.man.html#lbDK

The virtualOutbound in envoy will be hit. This is a special listener. It contains the Original Destination listener filter. Note "useOriginalDst": truethat after such configuration in the following configuration , envoy will re-find the matching listener in the configuration. If found, press the hit. The listener performs follow-up processing. If it cannot find it, it sends the request to the cluster in this listener. Here is a passthrough cluster. This cluster will forward the packet directly to the fourth layer.

```
root@k8s-master-v1-16 ~]# istioctl proxy-config listener productpage-v1-7f4cc988c6-qxqjs.istio-bookinfo --port 15001 -o json
&#91;
    {
        "name": "virtualOutbound",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 15001
            }
        },
        "filterChains": &#91;
            {
                "filters": &#91;
                    {
                        "name": "istio.stats",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
                            "value": {
                                "config": {
                                    "configuration": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n",
                                    "root_id": "stats_outbound",
                                    "vm_config": {
                                        "code": {
                                            "local": {
                                                "inline_string": "envoy.wasm.stats"
                                            }
                                        },
                                        "runtime": "envoy.wasm.runtime.null",
                                        "vm_id": "tcp_stats_outbound"
                                    }
                                }
                            }
                        }
                    },
                    {
                        "name": "envoy.tcp_proxy",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy",
                            "statPrefix": "PassthroughCluster",
                            "cluster": "PassthroughCluster",
                            "accessLog": &#91;
                                {
                                    "name": "envoy.file_access_log",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                        "path": "/dev/stdout",
                                        "format": "&#91;%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                    }
                                }
                            ]
                        }
                    }
                ],
                "name": "virtualOutbound-catchall-tcp"
            }
        ],
        "useOriginalDst": true,
        "trafficDirection": "OUTBOUND"
    }
]
```
The above listener will hand over the connection to the listener that matches the original destination IP and port. In the bookinfo example, it will be handed over to the 9080 listener. There is a question to consider here. From the perspective of envoy, the destination port of this link is already 15001, why can it match the following port 0.0.0.0:9080. This is because the NAT is done in the system kernel when iptables is redirected. The system kernel has this converted storage. Envoy obtains the real destination port through getsockopt() , so that it can correctly match the business listener.

```
&#91;root@k8s-master-v1-16 ~]# istioctl proxy-config listener productpage-v1-7f4cc988c6-qxqjs.istio-bookinfo --port 9080 -o json
&#91;
    {
        "name": "0.0.0.0_9080",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 9080
            }
        },
        "filterChains": &#91;
            {
                "filterChainMatch": {
                    "applicationProtocols": &#91;
                        "http/1.0",
                        "http/1.1",
                        "h2c"
                    ]
                },
                "filters": &#91;
                    {
                        "name": "envoy.http_connection_manager",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager",
                            "statPrefix": "outbound_0.0.0.0_9080",
                            "rds": {
                                "configSource": {
                                    "ads": {}
                                },
                                "routeConfigName": "9080"
                            },
```

```
&#91;root@k8s-master-v1-16 ~]# istioctl proxy-config listener productpage-v1-7f4cc988c6-qxqjs.istio-bookinfo
ADDRESS            PORT      TYPE
10.102.252.88      15012     TCP
10.96.122.225      31400     TCP
10.96.0.10         53        TCP
10.110.88.185      15443     TCP
10.96.122.225      15443     TCP
10.110.88.185      443       TCP
10.102.252.88      443       TCP
10.106.109.172     9001      TCP
10.96.0.1          443       TCP
10.96.122.225      443       TCP
10.106.109.172     9000      TCP
10.105.130.171     14267     HTTP+TCP
10.96.81.27        5601      HTTP+TCP
0.0.0.0            15014     HTTP+TCP
10.105.130.171     14250     HTTP+TCP
0.0.0.0            20001     HTTP+TCP
0.0.0.0            9411      HTTP+TCP
10.111.37.34       9090      HTTP+TCP
10.105.145.36      80        HTTP+TCP
10.105.130.171     14268     HTTP+TCP
10.96.0.10         9153      HTTP+TCP
10.110.244.188     80        HTTP+TCP
10.102.252.88      853       HTTP+TCP
10.99.139.251      16686     HTTP+TCP
0.0.0.0            12345     HTTP+TCP
10.103.155.149     80        HTTP+TCP
0.0.0.0            8000      HTTP+TCP
0.0.0.0            15010     HTTP+TCP
10.96.122.225      15020     HTTP+TCP
0.0.0.0            9090      HTTP+TCP
10.107.150.251     80        HTTP+TCP
0.0.0.0            14250     HTTP+TCP
0.0.0.0            80        HTTP+TCP
0.0.0.0            3000      HTTP+TCP
10.110.28.96       8181      HTTP+TCP
10.102.69.143      9200      HTTP+TCP
0.0.0.0            9080      HTTP+TCP 《《《《《《《《《《《《
0.0.0.0            15001     TCP 《《《《《《《《《《《《
0.0.0.0            15006     HTTP+TCP
0.0.0.0            15090     HTTP
0.0.0.0            15021     HTTP
```
Related passthrough cluster:

```
root@k8s-master-v1-16 ~]# istioctl proxy-config cluster productpage-v1-7f4cc988c6-qxqjs.istio-bookinfo --fqdn PassthroughCluster -o json
&#91;
。。。。忽略。。。
    {
        "name": "PassthroughCluster",
        "type": "ORIGINAL_DST",
        "connectTimeout": "10s",
        "lbPolicy": "CLUSTER_PROVIDED",
        "circuitBreakers": {
            "thresholds": &#91;
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295
                }
            ]
        },
        "filters": &#91;
            {
                "name": "istio.metadata_exchange",
                "typedConfig": {
                    "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                    "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                    "value": {
                        "protocol": "istio-peer-exchange"
                    }
                }
            }
        ]
    }
]
```

## Inbound flow control
When it is an inbound request, the destination address of the packet is the IP of the pod, and the destination port is the real port of the business (9080, non-svc mapping port). Since the link destination port of iptables is changed to 15006, it will Hit virtual inbound listener (0.0.0.0:15006), this listener has a series of filterchain, and the virtualoutbound listener configuration method is different, virtualinbound contains a series of actual service filters for specific ports, the connection will find specific in these fitlers Business matching. So how does it match the real 9080 business? For example, the following output:
addressPrefix can be matched. If the pod actually has multiple ports, only addressPrefix does not match. It also needs to match the application layer protocol, but the DestinationPort in the Match condition is not matched . In fact, it is similar to Virtualoutbound. The filter of the original destination listener is also used, so envoy will obtain the real destination port and IP from the kernel. This configuration method is different from the virtual outbound "useOriginalDst": true configuration method, because this is an updated configuration method." useOriginalDst": true This configuration is about to be abandoned.

`"listenerFilters": [ { "name": "envoy.listener.original_dst", "typedConfig": { "@type": "type.googleapis.com/envoy.config.filter.listener.original_dst.v2.OriginalDst" } },`

(According to https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/listener/listener_components.proto, the  filtermatch condition is that all must match)

```
            {
                "filterChainMatch": {
                    "destinationPort": 9080,
                    "prefixRanges": &#91;
                        {
                            "addressPrefix": "10.244.2.138",
                            "prefixLen": 32
                        }
                    ],
### 根据https://istio.io/latest/zh/docs/ops/deployment/requirements/ , 
### 同一个业务端口是不能被两个用不同协议的svc来发布的，因此这帮助避免了同端口同协议的match在整个配置文件里的出现。
                    "applicationProtocols": &#91;
                        "istio",
                        "istio-http/1.0",
                        "istio-http/1.1",
                        "istio-h2"
                    ]
                },
                "filters": &#91;
                    {
                        "name": "istio.metadata_exchange",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                            "value": {
                                "protocol": "istio-peer-exchange"
                            }
                        }
                    },
                    {
                        "name": "envoy.http_connection_manager",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager",
                            "statPrefix": "inbound_10.244.2.138_9080",
                            "routeConfig": {
                                "name": "inbound|9080|http|productpage.istio-bookinfo.svc.cluster.local",
                                "virtualHosts": &#91;
                                    {
                                        "name": "inbound|http|9080",
                                        "domains": &#91;
                                            "*"
                                        ],
                                        "routes": &#91;
                                            {
                                                "name": "default",
                                                "match": {
                                                    "prefix": "/"
                                                },
                                                "route": {
                                                    "cluster": "inbound|9080|http|productpage.istio-bookinfo.svc.cluster.local",
                                                    "timeout": "0s",
                                                    "maxGrpcTimeout": "0s"
                                                },
                                                "decorator": {
                                                    "operation": "productpage.istio-bookinfo.svc.cluster.local:9080/*"
                                                }
                                            }
                                        ]
                                    }
                                ],
```

## Last

Attach the actual configuration of 15006

```
&#91;root@k8s-master-v1-16 ~]# istioctl proxy-config listener productpage-v1-7f4cc988c6-qxqjs.istio-bookinfo --port 15006 -o json
&#91;
    {
        "name": "virtualInbound",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 15006
            }
        },
        "filterChains": &#91;
            {
                "filterChainMatch": {
                    "prefixRanges": &#91;
                        {
                            "addressPrefix": "0.0.0.0",
                            "prefixLen": 0
                        }
                    ],
                    "transportProtocol": "tls",
                    "applicationProtocols": &#91;
                        "istio-peer-exchange",
                        "istio"
                    ]
                },
                "filters": &#91;
                    {
                        "name": "istio.metadata_exchange",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                            "value": {
                                "protocol": "istio-peer-exchange"
                            }
                        }
                    },
                    {
                        "name": "istio.stats",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
                            "value": {
                                "config": {
                                    "configuration": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n",
                                    "root_id": "stats_inbound",
                                    "vm_config": {
                                        "code": {
                                            "local": {
                                                "inline_string": "envoy.wasm.stats"
                                            }
                                        },
                                        "runtime": "envoy.wasm.runtime.null",
                                        "vm_id": "tcp_stats_inbound"
                                    }
                                }
                            }
                        }
                    },
                    {
                        "name": "envoy.tcp_proxy",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy",
                            "statPrefix": "InboundPassthroughClusterIpv4",
                            "cluster": "InboundPassthroughClusterIpv4",
                            "accessLog": &#91;
                                {
                                    "name": "envoy.file_access_log",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                        "path": "/dev/stdout",
                                        "format": "&#91;%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                    }
                                }
                            ]
                        }
                    }
                ],
                "transportSocket": {
                    "name": "envoy.transport_sockets.tls",
                    "typedConfig": {
                        "@type": "type.googleapis.com/envoy.api.v2.auth.DownstreamTlsContext",
                        "commonTlsContext": {
                            "tlsCertificateSdsSecretConfigs": &#91;
                                {
                                    "name": "default",
                                    "sdsConfig": {
                                        "apiConfigSource": {
                                            "apiType": "GRPC",
                                            "grpcServices": &#91;
                                                {
                                                    "envoyGrpc": {
                                                        "clusterName": "sds-grpc"
                                                    }
                                                }
                                            ]
                                        }
                                    }
                                }
                            ],
                            "combinedValidationContext": {
                                "defaultValidationContext": {},
                                "validationContextSdsSecretConfig": {
                                    "name": "ROOTCA",
                                    "sdsConfig": {
                                        "apiConfigSource": {
                                            "apiType": "GRPC",
                                            "grpcServices": &#91;
                                                {
                                                    "envoyGrpc": {
                                                        "clusterName": "sds-grpc"
                                                    }
                                                }
                                            ]
                                        }
                                    }
                                }
                            },
                            "alpnProtocols": &#91;
                                "istio-peer-exchange",
                                "h2",
                                "http/1.1"
                            ]
                        },
                        "requireClientCertificate": true
                    }
                },
                "name": "virtualInbound"
            },
            {
                "filterChainMatch": {
                    "prefixRanges": &#91;
                        {
                            "addressPrefix": "0.0.0.0",
                            "prefixLen": 0
                        }
                    ]
                },
                "filters": &#91;
                    {
                        "name": "istio.metadata_exchange",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                            "value": {
                                "protocol": "istio-peer-exchange"
                            }
                        }
                    },
                    {
                        "name": "istio.stats",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
                            "value": {
                                "config": {
                                    "configuration": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n",
                                    "root_id": "stats_inbound",
                                    "vm_config": {
                                        "code": {
                                            "local": {
                                                "inline_string": "envoy.wasm.stats"
                                            }
                                        },
                                        "runtime": "envoy.wasm.runtime.null",
                                        "vm_id": "tcp_stats_inbound"
                                    }
                                }
                            }
                        }
                    },
                    {
                        "name": "envoy.tcp_proxy",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy",
                            "statPrefix": "InboundPassthroughClusterIpv4",
                            "cluster": "InboundPassthroughClusterIpv4",
                            "accessLog": &#91;
                                {
                                    "name": "envoy.file_access_log",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                        "path": "/dev/stdout",
                                        "format": "&#91;%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                    }
                                }
                            ]
                        }
                    }
                ],
                "name": "virtualInbound"
            },
            {
                "filterChainMatch": {
                    "prefixRanges": &#91;
                        {
                            "addressPrefix": "0.0.0.0",
                            "prefixLen": 0
                        }
                    ],
                    "transportProtocol": "tls",
                    "applicationProtocols": &#91;
                        "http/1.0",
                        "http/1.1",
                        "h2c",
                        "istio-http/1.0",
                        "istio-http/1.1",
                        "istio-h2"
                    ]
                },
                "filters": &#91;
                    {
                        "name": "istio.metadata_exchange",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                            "value": {
                                "protocol": "istio-peer-exchange"
                            }
                        }
                    },
                    {
                        "name": "envoy.http_connection_manager",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager",
                            "statPrefix": "InboundPassthroughClusterIpv4",
                            "routeConfig": {
                                "name": "InboundPassthroughClusterIpv4",
                                "virtualHosts": &#91;
                                    {
                                        "name": "inbound|http|0",
                                        "domains": &#91;
                                            "*"
                                        ],
                                        "routes": &#91;
                                            {
                                                "name": "default",
                                                "match": {
                                                    "prefix": "/"
                                                },
                                                "route": {
                                                    "cluster": "InboundPassthroughClusterIpv4",
                                                    "timeout": "0s",
                                                    "maxGrpcTimeout": "0s"
                                                },
                                                "decorator": {
                                                    "operation": ":0/*"
                                                }
                                            }
                                        ]
                                    }
                                ],
                                "validateClusters": false
                            },
                            "httpFilters": &#91;
                                {
                                    "name": "istio.metadata_exchange",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": "{}\n",
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.metadata_exchange"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.cors",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.cors.v2.Cors"
                                    }
                                },
                                {
                                    "name": "envoy.fault",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.fault.v2.HTTPFault"
                                    }
                                },
                                {
                                    "name": "istio.stats",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n",
                                                "root_id": "stats_inbound",
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.stats"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null",
                                                    "vm_id": "stats_inbound"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.router",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.router.v2.Router"
                                    }
                                }
                            ],
                            "tracing": {
                                "clientSampling": {
                                    "value": 100
                                },
                                "randomSampling": {
                                    "value": 100
                                },
                                "overallSampling": {
                                    "value": 100
                                }
                            },
                            "serverName": "istio-envoy",
                            "streamIdleTimeout": "0s",
                            "accessLog": &#91;
                                {
                                    "name": "envoy.file_access_log",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                        "path": "/dev/stdout",
                                        "format": "&#91;%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                    }
                                }
                            ],
                            "useRemoteAddress": false,
                            "generateRequestId": true,
                            "forwardClientCertDetails": "APPEND_FORWARD",
                            "setCurrentClientCertDetails": {
                                "subject": true,
                                "dns": true,
                                "uri": true
                            },
                            "upgradeConfigs": &#91;
                                {
                                    "upgradeType": "websocket"
                                }
                            ],
                            "normalizePath": true
                        }
                    }
                ],
                "transportSocket": {
                    "name": "envoy.transport_sockets.tls",
                    "typedConfig": {
                        "@type": "type.googleapis.com/envoy.api.v2.auth.DownstreamTlsContext",
                        "commonTlsContext": {
                            "tlsCertificateSdsSecretConfigs": &#91;
                                {
                                    "name": "default",
                                    "sdsConfig": {
                                        "apiConfigSource": {
                                            "apiType": "GRPC",
                                            "grpcServices": &#91;
                                                {
                                                    "envoyGrpc": {
                                                        "clusterName": "sds-grpc"
                                                    }
                                                }
                                            ]
                                        }
                                    }
                                }
                            ],
                            "combinedValidationContext": {
                                "defaultValidationContext": {},
                                "validationContextSdsSecretConfig": {
                                    "name": "ROOTCA",
                                    "sdsConfig": {
                                        "apiConfigSource": {
                                            "apiType": "GRPC",
                                            "grpcServices": &#91;
                                                {
                                                    "envoyGrpc": {
                                                        "clusterName": "sds-grpc"
                                                    }
                                                }
                                            ]
                                        }
                                    }
                                }
                            },
                            "alpnProtocols": &#91;
                                "h2",
                                "http/1.1"
                            ]
                        },
                        "requireClientCertificate": true
                    }
                },
                "name": "virtualInbound-catchall-http"
            },
            {
                "filterChainMatch": {
                    "prefixRanges": &#91;
                        {
                            "addressPrefix": "0.0.0.0",
                            "prefixLen": 0
                        }
                    ],
                    "applicationProtocols": &#91;
                        "http/1.0",
                        "http/1.1",
                        "h2c"
                    ]
                },
                "filters": &#91;
                    {
                        "name": "istio.metadata_exchange",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                            "value": {
                                "protocol": "istio-peer-exchange"
                            }
                        }
                    },
                    {
                        "name": "envoy.http_connection_manager",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager",
                            "statPrefix": "InboundPassthroughClusterIpv4",
                            "routeConfig": {
                                "name": "InboundPassthroughClusterIpv4",
                                "virtualHosts": &#91;
                                    {
                                        "name": "inbound|http|0",
                                        "domains": &#91;
                                            "*"
                                        ],
                                        "routes": &#91;
                                            {
                                                "name": "default",
                                                "match": {
                                                    "prefix": "/"
                                                },
                                                "route": {
                                                    "cluster": "InboundPassthroughClusterIpv4",
                                                    "timeout": "0s",
                                                    "maxGrpcTimeout": "0s"
                                                },
                                                "decorator": {
                                                    "operation": ":0/*"
                                                }
                                            }
                                        ]
                                    }
                                ],
                                "validateClusters": false
                            },
                            "httpFilters": &#91;
                                {
                                    "name": "istio.metadata_exchange",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": "{}\n",
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.metadata_exchange"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.cors",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.cors.v2.Cors"
                                    }
                                },
                                {
                                    "name": "envoy.fault",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.fault.v2.HTTPFault"
                                    }
                                },
                                {
                                    "name": "istio.stats",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n",
                                                "root_id": "stats_inbound",
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.stats"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null",
                                                    "vm_id": "stats_inbound"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.router",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.router.v2.Router"
                                    }
                                }
                            ],
                            "tracing": {
                                "clientSampling": {
                                    "value": 100
                                },
                                "randomSampling": {
                                    "value": 100
                                },
                                "overallSampling": {
                                    "value": 100
                                }
                            },
                            "serverName": "istio-envoy",
                            "streamIdleTimeout": "0s",
                            "accessLog": &#91;
                                {
                                    "name": "envoy.file_access_log",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                        "path": "/dev/stdout",
                                        "format": "&#91;%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                    }
                                }
                            ],
                            "useRemoteAddress": false,
                            "generateRequestId": true,
                            "forwardClientCertDetails": "APPEND_FORWARD",
                            "setCurrentClientCertDetails": {
                                "subject": true,
                                "dns": true,
                                "uri": true
                            },
                            "upgradeConfigs": &#91;
                                {
                                    "upgradeType": "websocket"
                                }
                            ],
                            "normalizePath": true
                        }
                    }
                ],
                "name": "virtualInbound-catchall-http"
            },
            {
                "filterChainMatch": {
                    "destinationPort": 15021,
                    "prefixRanges": &#91;
                        {
                            "addressPrefix": "10.244.2.155",
                            "prefixLen": 32
                        }
                    ]
                },
                "filters": &#91;
                    {
                        "name": "istio.metadata_exchange",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                            "value": {
                                "protocol": "istio-peer-exchange"
                            }
                        }
                    },
                    {
                        "name": "istio.stats",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
                            "value": {
                                "config": {
                                    "configuration": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n",
                                    "root_id": "stats_inbound",
                                    "vm_config": {
                                        "code": {
                                            "local": {
                                                "inline_string": "envoy.wasm.stats"
                                            }
                                        },
                                        "runtime": "envoy.wasm.runtime.null",
                                        "vm_id": "tcp_stats_inbound"
                                    }
                                }
                            }
                        }
                    },
                    {
                        "name": "envoy.tcp_proxy",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy",
                            "statPrefix": "inbound|15021|mgmt-15021|mgmtCluster",
                            "cluster": "inbound|15021|mgmt-15021|mgmtCluster",
                            "accessLog": &#91;
                                {
                                    "name": "envoy.file_access_log",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                        "path": "/dev/stdout",
                                        "format": "&#91;%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                    }
                                }
                            ]
                        }
                    }
                ],
                "name": "10.244.2.155_15021"
            },
            {
                "filterChainMatch": {
                    "destinationPort": 9080,
                    "prefixRanges": &#91;
                        {
                            "addressPrefix": "10.244.2.155",
                            "prefixLen": 32
                        }
                    ],
                    "applicationProtocols": &#91;
                        "istio",
                        "istio-http/1.0",
                        "istio-http/1.1",
                        "istio-h2"
                    ]
                },
                "filters": &#91;
                    {
                        "name": "istio.metadata_exchange",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                            "value": {
                                "protocol": "istio-peer-exchange"
                            }
                        }
                    },
                    {
                        "name": "envoy.http_connection_manager",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager",
                            "statPrefix": "inbound_10.244.2.155_9080",
                            "routeConfig": {
                                "name": "inbound|9080|http|productpage.istio-bookinfo.svc.cluster.local",
                                "virtualHosts": &#91;
                                    {
                                        "name": "inbound|http|9080",
                                        "domains": &#91;
                                            "*"
                                        ],
                                        "routes": &#91;
                                            {
                                                "name": "default",
                                                "match": {
                                                    "prefix": "/"
                                                },
                                                "route": {
                                                    "cluster": "inbound|9080|http|productpage.istio-bookinfo.svc.cluster.local",
                                                    "timeout": "0s",
                                                    "maxGrpcTimeout": "0s"
                                                },
                                                "decorator": {
                                                    "operation": "productpage.istio-bookinfo.svc.cluster.local:9080/*"
                                                }
                                            }
                                        ]
                                    }
                                ],
                                "validateClusters": false
                            },
                            "httpFilters": &#91;
                                {
                                    "name": "istio.metadata_exchange",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": "{}\n",
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.metadata_exchange"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "istio_authn",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/istio.envoy.config.filter.http.authn.v2alpha1.FilterConfig",
                                        "policy": {
                                            "peers": &#91;
                                                {
                                                    "mtls": {
                                                        "mode": "PERMISSIVE"
                                                    }
                                                }
                                            ]
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.cors",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.cors.v2.Cors"
                                    }
                                },
                                {
                                    "name": "envoy.fault",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.fault.v2.HTTPFault"
                                    }
                                },
                                {
                                    "name": "istio.stats",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n",
                                                "root_id": "stats_inbound",
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.stats"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null",
                                                    "vm_id": "stats_inbound"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.router",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.router.v2.Router"
                                    }
                                }
                            ],
                            "tracing": {
                                "clientSampling": {
                                    "value": 100
                                },
                                "randomSampling": {
                                    "value": 100
                                },
                                "overallSampling": {
                                    "value": 100
                                }
                            },
                            "serverName": "istio-envoy",
                            "streamIdleTimeout": "0s",
                            "accessLog": &#91;
                                {
                                    "name": "envoy.file_access_log",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                        "path": "/dev/stdout",
                                        "format": "&#91;%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                    }
                                }
                            ],
                            "useRemoteAddress": false,
                            "generateRequestId": true,
                            "forwardClientCertDetails": "APPEND_FORWARD",
                            "setCurrentClientCertDetails": {
                                "subject": true,
                                "dns": true,
                                "uri": true
                            },
                            "upgradeConfigs": &#91;
                                {
                                    "upgradeType": "websocket"
                                }
                            ],
                            "normalizePath": true
                        }
                    }
                ],
                "transportSocket": {
                    "name": "envoy.transport_sockets.tls",
                    "typedConfig": {
                        "@type": "type.googleapis.com/envoy.api.v2.auth.DownstreamTlsContext",
                        "commonTlsContext": {
                            "tlsCertificateSdsSecretConfigs": &#91;
                                {
                                    "name": "default",
                                    "sdsConfig": {
                                        "apiConfigSource": {
                                            "apiType": "GRPC",
                                            "grpcServices": &#91;
                                                {
                                                    "envoyGrpc": {
                                                        "clusterName": "sds-grpc"
                                                    }
                                                }
                                            ]
                                        }
                                    }
                                }
                            ],
                            "combinedValidationContext": {
                                "defaultValidationContext": {},
                                "validationContextSdsSecretConfig": {
                                    "name": "ROOTCA",
                                    "sdsConfig": {
                                        "apiConfigSource": {
                                            "apiType": "GRPC",
                                            "grpcServices": &#91;
                                                {
                                                    "envoyGrpc": {
                                                        "clusterName": "sds-grpc"
                                                    }
                                                }
                                            ]
                                        }
                                    }
                                }
                            },
                            "alpnProtocols": &#91;
                                "h2",
                                "http/1.1"
                            ]
                        },
                        "requireClientCertificate": true
                    }
                },
                "name": "10.244.2.155_9080"
            },
            {
                "filterChainMatch": {
                    "destinationPort": 9080,
                    "prefixRanges": &#91;
                        {
                            "addressPrefix": "10.244.2.155",
                            "prefixLen": 32
                        }
                    ]
                },
                "filters": &#91;
                    {
                        "name": "istio.metadata_exchange",
                        "typedConfig": {
                            "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                            "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                            "value": {
                                "protocol": "istio-peer-exchange"
                            }
                        }
                    },
                    {
                        "name": "envoy.http_connection_manager",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager",
                            "statPrefix": "inbound_10.244.2.155_9080",
                            "routeConfig": {
                                "name": "inbound|9080|http|productpage.istio-bookinfo.svc.cluster.local",
                                "virtualHosts": &#91;
                                    {
                                        "name": "inbound|http|9080",
                                        "domains": &#91;
                                            "*"
                                        ],
                                        "routes": &#91;
                                            {
                                                "name": "default",
                                                "match": {
                                                    "prefix": "/"
                                                },
                                                "route": {
                                                    "cluster": "inbound|9080|http|productpage.istio-bookinfo.svc.cluster.local",
                                                    "timeout": "0s",
                                                    "maxGrpcTimeout": "0s"
                                                },
                                                "decorator": {
                                                    "operation": "productpage.istio-bookinfo.svc.cluster.local:9080/*"
                                                }
                                            }
                                        ]
                                    }
                                ],
                                "validateClusters": false
                            },
                            "httpFilters": &#91;
                                {
                                    "name": "istio.metadata_exchange",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": "{}\n",
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.metadata_exchange"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "istio_authn",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/istio.envoy.config.filter.http.authn.v2alpha1.FilterConfig",
                                        "policy": {
                                            "peers": &#91;
                                                {
                                                    "mtls": {
                                                        "mode": "PERMISSIVE"
                                                    }
                                                }
                                            ]
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.cors",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.cors.v2.Cors"
                                    }
                                },
                                {
                                    "name": "envoy.fault",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.fault.v2.HTTPFault"
                                    }
                                },
                                {
                                    "name": "istio.stats",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n",
                                                "root_id": "stats_inbound",
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.stats"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null",
                                                    "vm_id": "stats_inbound"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.router",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.filter.http.router.v2.Router"
                                    }
                                }
                            ],
                            "tracing": {
                                "clientSampling": {
                                    "value": 100
                                },
                                "randomSampling": {
                                    "value": 100
                                },
                                "overallSampling": {
                                    "value": 100
                                }
                            },
                            "serverName": "istio-envoy",
                            "streamIdleTimeout": "0s",
                            "accessLog": &#91;
                                {
                                    "name": "envoy.file_access_log",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                        "path": "/dev/stdout",
                                        "format": "&#91;%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                    }
                                }
                            ],
                            "useRemoteAddress": false,
                            "generateRequestId": true,
                            "forwardClientCertDetails": "APPEND_FORWARD",
                            "setCurrentClientCertDetails": {
                                "subject": true,
                                "dns": true,
                                "uri": true
                            },
                            "upgradeConfigs": &#91;
                                {
                                    "upgradeType": "websocket"
                                }
                            ],
                            "normalizePath": true
                        }
                    }
                ],
                "name": "10.244.2.155_9080"
            }
        ],
        "listenerFilters": &#91;
            {
                "name": "envoy.listener.original_dst",
                "typedConfig": {
                    "@type": "type.googleapis.com/envoy.config.filter.listener.original_dst.v2.OriginalDst"
                }
            },
            {
                "name": "envoy.listener.tls_inspector",
                "typedConfig": {
                    "@type": "type.googleapis.com/envoy.config.filter.listener.tls_inspector.v2.TlsInspector"
                }
            },
            {
                "name": "envoy.listener.http_inspector",
                "typedConfig": {
                    "@type": "type.googleapis.com/envoy.config.filter.listener.http_inspector.v2.HttpInspector"
                }
            }
        ],
        "listenerFiltersTimeout": "1s",
        "continueOnListenerFiltersTimeout": true,
        "trafficDirection": "INBOUND"
    }
]
```

Check more istio practice detail at my tech blog https://cnadn.net