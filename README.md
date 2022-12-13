<p align="center">
  <img src="imgs/vegvisir_banner_large_whitefont.png" height="250px"/>
</p>


# Minimal readme version
Configuration structures are defined using the [Concise Data Definition Language](https://www.rfc-editor.org/rfc/rfc8610).

**Nice to knows**

- Log paths have no trailing slashes
- Variable syntax: !{VAR}
  - Regex: !{[A-Z0-9_-]+}
- Vegvisir requires two files to boot and to run experiments; Configuration files are (for now) passed as arguments to the python script
  1. An *implementations* file
  2. An *experiment* file

# Implementations
Contains a list of all implementations and their configurations.

```
Implementations = {
  clients: {
    + ImplementationName: ClientImplementation
  },
  servers: {
    + ImplementationName: ServerImplementation
  },
  shapers: {
    + ImplementationName: ShaperImplementation
  },
}
ImplementationName = text .regex "TODO no spaces or special chars"
```

```
ClientImplementation = {
  ClientType
  ? parameters: Parameter
  ? construct : [* CDCommand] ; Commands executed before "command" if the implementation represents a host command
  ? destruct : [* CDCommand] ; Commands executed before "command" if the implementation represents a host command
}
```

```
ClientType = (
  image: text ; repo/name:tag //
  command: text ; host command
)
```

```
ServerImplementation = {
  ? image: text ; repo/name:tag
  ? parameters: Parameter
}
```

```
ShaperImplementation = {
  ? image: text ; repo/name:tag
  ? parameters: Parameter
}
```

```
Parameter = {
  * ParameterKey => text
}
ParameterKey = text .regex "\!(?:(?:\{(?P<parameter>(?:[A-Z0-9_-]+))\})"
```

```
CDCommand = {
  ? root_required: bool .default false
  command: text
}
```

# Experiments
Contains a single (repeatable) experiment

```
Experiment = {
  clients: [ClientExperiment]
  servers: [ServerExperiment]
  shapers: [ShaperExperiment]
  environment : Environment
  settings: Settings
}
```

```
CLientExperiment = {
  name: text ; must match one of the names from implementations
  ? log_name : text ; Name used for logging, if the same client is used for multiple experiment permutation (e.g., other arguments), a unique log_name must be provided for all entries
  ? arguments : Arguments
}
```

```
Arguments = {
  * (text .regex "\!(?:(?:\{(?P<parameter>(?:[A-Z0-9_-]+))\})") => text ; Matches a parameter of the respective implementation configuration
}
```

```
Environment = {
  name: DefaultEnvironments
  ? sensors : [SensorConfiguration]
}
```

```
DefaultEnvironments = "webserver-basic" ; Currently only 1 available
```

```
SensorConfiguration = {
  name: AvailableSensors
  * SensorKey => any
}
SensorKey = text ; Parameter as defined in the sensor python code
```

```
AvailableSensors = "timeout" / "file-watchdog"
```

```
Settings = {
  label: text .regex "TODO (no spaces or weird folder breaking names)" ; label in the logging output folder
  ? description : text ; Currently unused, but optional to describe the setup in the configuration file
  ? playground : bool .default false ; Currently unused, coming soon
  ? www_dir : text .default "./www" ; Web root path
  ? iterations: int .default 1 ; The number of times the complete permutation needs to be repeated
}
```

# JSON examples
## implementations.json

```
{
	"clients": {
		"test": {
			"command": "echo Hello I am a command my input is !{INPUT}",
			"parameters":{
				"INPUT": true
			},
			"construct": [
				{
					"root_required": true,
					"command": "whoami"
				},
				{
					"root_required": false,
					"command": "touch \"!{LOG_PATH_CLIENT}/hello_world.txt\""
				},
				{
					"root_required": false,
					"command": "echo \"!{INPUT}\" > !{LOG_PATH_CLIENT}/hello_world.txt"
				}
			],
			"destruct": [
				{
					"root_required": true,
					"command": "whoami"
				},
				{
					"command": "echo ' | Destroyed :)' >> !{LOG_PATH_CLIENT}/hello_world.txt"
				}
			]
		},

		"aioquic": {
			"image": "aiortc/aioquic-qns",
			"parameters": {
				"REQUESTS": true
			}
		}
	},

	"shapers": {
		"ns3-quic": {
			"image": "martenseemann/quic-network-simulator",
			"scenarios": {
				"simple-p2p": "\"simple-p2p --delay=15ms --bandwidth=10Mbps --queue=25\""
			}
		},
		"tc-netem": {
			"image": "tc-netem",
			"scenarios": {
				"simple": {
					"command": "\"simple {LATENCY} {THROUGHPUT}\"",
					"parameters": ["THROUGHPUT", "LATENCY"]
				},
				"simple-15-10": "\"simple 15 10\"",
				"cellular-loss-median": "\"akamai_cellular_emulation.sh loss_based median\"",
				"cellular-experience-fair": "\"akamai_cellular_emulation.sh experience_based fair\""
			}
		}
	},

	"servers": {
		"aioquic": {
			"image": "aiortc/aioquic-qns",
			"debug": "aioquic-{EXAMPLE}"
		},
		"quic-go": {
			"image": "martenseemann/quic-go-interop:latest"
		},
		"quicly": {
			"image": "h2oserver/quicly-interop-runner:latest"
		},
		"ngtcp2": {
			"image": "ghcr.io/ngtcp2/ngtcp2-interop:latest"
		},
		"quant": {
			"image": "ntap/quant:interop"
		},
		"mvfst": {
			"image": "lnicco/mvfst-qns:latest"
		},
		"quiche": {
			"image": "cloudflare/quiche-qns:latest"
		},
		"kwik": {
			"image": "peterdoornbosch/kwik_n_flupke-interop"
		},
		"picoquic": {
			"image": "privateoctopus/picoquic:latest"
		},
		"neqo": {
			"image": "neqoquic/neqo-qns:latest"
		},
		"nginx": {
			"image": "nginx/nginx-quic-qns:latest"
		},
		"msquic": {
			"image": "mcr.microsoft.com/msquic/qns:latest"
		},
		"xquic": {
			"image": "kulsk/xquic:latest"
		},
		"lsquic": {
			"image": "litespeedtech/lsquic-qir:latest"
		}
	}
}
```

## experiment1.json
```
{
    "clients": [
        {
            "name": "test",
            "arguments": {
                "INPUT": "https://!{ORIGIN}/funny_url/!!{INPUT}"
            }
        }
    ],

    "shapers": [
        {
            "name": "ns3-quic",
            "scenario": "simple-p2p"
        }
    ],
    
    "servers": [
        {
            "name": "aioquic"
        }
    ],

    "environment": {
        "name": "webserver-basic",
        "sensors": [
            {
                "name": "timeout",
                "timeout": 5
            }
        ]
    },

    "settings": {
        "label": "host_command_tests",
        "description": "", 
        "playground": false,
        "www_dir": "./www",
        "iterations": 1
    }
}
```

## experiment2.json
```
{
    "clients": [
        {
            "name": "aioquic",
            "arguments": {
                "REQUESTS": "https://!{ORIGIN}/1gb.bin"
            }
        },
        {
            "name": "aioquic",
            "log_name": "aioquic2",
            "arguments": {
                "REQUESTS": "https://!{ORIGIN}/1gb.bin"
            }
        }
    ],

    "shapers": [
        {
            "name": "ns3-quic",
            "scenario": "simple-p2p"
        }, 
        {
            "name": "tc-netem",
            "scenario": "simple-15-10"
        }
    ],
    
    "servers": [
        {
            "name": "aioquic"
        }
    ],

    "environment": {
        "name": "webserver-basic",
        "sensors": [
            {
                "name": "timeout",
                "timeout": 5
            }
        ]
    },

    "settings": {
        "label": "docker-to-docker-1gb",
        "description": "", 
        "playground": false,
        "www_dir": "./www",
        "iterations": 1
    }
}
```