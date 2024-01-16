# Launching Self-Hosted Scramjet Transform Hub on OpenMetal Cloud platform

## Objective

This guide provides a step-by-step guide designed to facilitate the setup and launch of the Scramjet Transform Hub on Openstack utilizing Heat Orchestration. Once initiated, the hub will seamlessly connect to the user's environment within the Scramjet Cloud Platform as a Self-Hosted Hub.

## What is Scramjet Cloud Platform?

The Scramjet Cloud Platform stands as a robust platform designed for distributed data processing. It simplifies the deployment and execution of data integration tasks across diverse locations and environments. This is achieved by reducing the need for extensive boilerplate code and intricate configurations, thereby streamlining the process for users.

Read more: https://scramjet.org/cloud-platform/

## What can I do with Scramjet Transform Hub?

Scramjet Transform Hub is a cornerstone of the Scramjet Cloud Platform, offering:

- Data Integration: Easily connect data from diverse sources like on-premise servers, eliminating the challenges of VPNs or firewall configurations.
- Adaptive Deployment: Run programs using versatile adapters like Kubernetes, Docker, or direct processes.
- Stream-Based Processing: Dynamically manage data flows between functions in sequences.
- Centralized Management: With Self Hosted Hub, integrate on-premise hubs into the Scramjet Cloud Platform for unified oversight.
- Deployment Across Platforms: Launch programs effortlessly on remote servers or various cloud providers. Leverage its capabilities for efficient and comprehensive data processing and deployment.

Read more: https://docs.scramjet.org/platform/transform-hub/

## Prerequisites
- Configured project in OpenMetal Cloud: An active project set up within the OpenMetal cloud environment - https://openmetal.io/docs/manuals/operators-manual/day-1/horizon/getting-started-with-horizon
- Scramjet Cloud Platform Account: A registered and active account on the Scramjet Cloud Platform and configured CLI - https://docs.scramjet.org/platform/get-started/

## Deploying Self-Hosted Hub using Openstack Heat

### Generate an Access Key

Create an Access Key to establish a connection between your Self Hosted Hubs and the Scramjet Cloud Platform. This can be done using the SCP Panel or CLI:

```bash
si space access create <description>

# Example: si space access create openmetal
```
Write down returned `accessKey`

### Self-Hosted Hub Heat Orchestration Template

The following showcases a Heat Orchestration Template (HOT) to deploy an instance of a Self-Hosted Hub on a Debian/Ubuntu-like system.

This manifest establishes two key resources:

- An OpenStack server instance.
- A Heat software configuration. The associated `user_data` includes a script that installs essential dependencies, sets up the Scramjet Transform Hub, and seamlessly links it to the user's Space within the Scramjet Cloud Platform.

Self-Hosted Hub Heat Orchestration Template:

```bash
heat_template_version: 2018-03-02

description: Simple HOT to deploy a single Self-Hosted Hub instance and connect it to Scramjet Cloud Platform

parameters:
  key_name:
    type: string
    label: Key Name
    description: Name of key-pair to be used for compute instance
  image_id:
    type: string
    label: Image ID
    description: Image to be used for compute instance (Debian/Ubuntu like)
  instance_type:
    type: string
    label: Instance Type
    description: Type of instance (flavor) to be used
  network_name:
    type: string
    description: The network to be used
  security_groups:
    type: comma_delimited_list
    description: Name of the Security Group
  scp_accesskey:
    type: string
    description: Scramjet Cloud Platform Space Access key
    hidden: true
  scp_spaceid:
    type: string
    description: Scramjet Cloud Platform Space ID
    hidden: true
  sth_description:
    type: string
    description: Scramjet Transform Hub instance description
    default: STH/OpenMetal instance created by Heat 
  sth_tags:
    type: string
    description: Scramjet Transform Hub instance tags (comma-separated)
    default: "openmetal"
resources:
  sth_user_data:
    type: OS::Heat::SoftwareConfig
    properties:
      group: script
      config: |
        #!/bin/sh

        set -e

        main() {
          NODE_VERSION="18.x"
          USER_HOME="/var/lib/sth"
          STH_CONFIG_FILE="$USER_HOME/config.json"
        
          # Update package list and install necessary packages
          apt-get update
          apt-get install -y curl software-properties-common python3-pip jq
        
          # Install Node.js and npm
          curl -fsSL https://deb.nodesource.com/gpgkey/nodesource.gpg.key | gpg --dearmor -o /usr/share/keyrings/nodesource-archive-keyring.gpg
          echo "deb [signed-by=/usr/share/keyrings/nodesource-archive-keyring.gpg] https://deb.nodesource.com/node_${NODE_VERSION} $(lsb_release -cs) main" > /etc/apt/sources.list.d/nodesource.list
          apt-get update
          apt-get install -y nodejs
        
          # Install Docker
          apt-get install -y docker.io
          systemctl enable --now docker
        
          # Install Python 3
          apt-get install -y python3
        
          # Install Scramjet Transform Hub
          npm install -g @scramjet/sth
        
          # Create a sth system user and group
          groupadd sth
          useradd -m --system --home-dir $USER_HOME --shell /bin/false -g sth sth
        
          # Add the user to the Docker group
          usermod -aG docker sth
        
          # Set STH id to instance name
          host_id=$(curl -s http://169.254.169.254/openstack/2020-10-14/meta_data.json | jq -r .name)
          
          # Get STH description from instance metadata
          sth_description=$(curl -s http://169.254.169.254/openstack/2020-10-14/meta_data.json | jq -r .meta.sth_description)
        
          # Get SCP access key from instance metadata
          scp_accesskey=$(curl -s http://169.254.169.254/openstack/2020-10-14/meta_data.json | jq -r .meta.scp_accesskey)

          # Get SCP space id from instance metadata
          scp_spaceid=$(curl -s http://169.254.169.254/openstack/2020-10-14/meta_data.json | jq -r .meta.scp_spaceid)
        
          # Get STH tags from instance metadata
          sth_tags=$(curl -s http://169.254.169.254/openstack/2020-10-14/meta_data.json | jq -r .meta.sth_tags)
        
          # Process STH tags to JSON values
          tags=$(IFS=","; for t in ${sth_tags}; do echo "\"${t}\","; done|sed '$s/,//')
        
          # Create a STH configuration file
          cat > $STH_CONFIG_FILE <<EOL
        {
            "host": {
                "hostname": "0.0.0.0",
                "port": 9002,
                "instancesServerPort": 9003,
                "id": "${host_id}"
            },
            "description": "${sth_description}",
            "tags": [
              ${tags}
            ],
            "timings": {
                "instanceLifetimeExtensionDelay": 10000
            },
            "platform": {
                "apiKey": "${scp_accesskey}",
                "api": "https://api-shh.scramjet.cloud",
                "space": "${scp_spaceid}"
            }
        }
        EOL
          
          # Set permissions to ensure the file is owned by the 'sth' user and not world-readable
          chown sth:sth ${STH_CONFIG_FILE}
          chmod 600 ${STH_CONFIG_FILE}
        
          # Create a systemd service file
          cat > /etc/systemd/system/scramjet-transform-hub.service <<EOL
        [Unit]
        Description=Scramjet Transform Hub
        After=network.target
        
        [Service]
        User=sth
        WorkingDirectory=${USER_HOME}
        ExecStart=/usr/bin/scramjet-transform-hub --config ${USER_HOME}/config.json
        Restart=always
        RestartSec=10
        StandardOutput=syslog
        StandardError=syslog
        SyslogIdentifier=scramjet-transform-hub
        
        [Install]
        WantedBy=multi-user.target
        EOL
        
          # Reload systemd and start the service
          systemctl daemon-reload
          systemctl enable scramjet-transform-hub
          systemctl start scramjet-transform-hub
        
          # Ensure the service is running
          systemctl status scramjet-transform-hub
        }
        
        # Call the main function
        main
  sth_instance:
    type: OS::Nova::Server
    properties:
      key_name: { get_param: key_name }
      image: { get_param: image_id }
      networks:
        - network: { get_param: network_name }
      flavor: { get_param: instance_type }
      security_groups: { get_param: security_groups }
      metadata: {
        scp_accesskey: { get_param: scp_accesskey },
        scp_spaceid: { get_param: scp_spaceid },
        sth_description: { get_param: sth_description },
        sth_tags: { get_param: sth_tags }
      }

      user_data_format: RAW
      user_data: { get_resource: sth_user_data }
outputs:
  server_ip:
    description: IP address of the created server instance
    value: { 
      get_attr: [
        sth_instance,
        first_address
      ] 
    }
```
#### Parameters

- `scp_accesskey` - Required parameter, Access Key generated in the previous step
- `scp_spaceid` - Required parameter in the format: <OrganizationId:SpaceId>. This parameter can be obtained using the Scramjet CLI:

```bash
si config set log --format "json"
si space info | jq -r .spaceId | sed 's/\(.*\)-[^-]*$/\1:&/'
```

- `sth_description` - Optional parameter, adds a description to provisioned Self-Hosted Hub, e.g. "STH/OpenMetal instance created by Heat"
- `sth_tags` - Optional parameter, adds comma-separated tags to provisioned Self-Hosted Hub, e.g. "openmetal,medium"

### Deploying a Heat Template in Horizon

To deploy a Heat Template using the Horizon dashboard, follow the detailed instructions provided in the OpenMetal Operators Manual - https://openmetal.io/docs/manuals/operators-manual/day-4/automation/heat. This resource offers step-by-step guidance to ensure a successful deployment of your Heat Template via the Horizon.

### Verifying the Self-Hosted Hub Provisioning

After successful deploying the Heat Template, let's confirm that the Self-Hosted Hub has been provisioned correctly and is functional. Use the following steps with Scramjet CLI:

#### 1. List Hubs in the user's Space:

Self-Hosted Hub should appear in the list with an healthy and connected status:

```bash
si hub ls | jq

...
{
  "id": "sth-instance-sth_instance-2vd6hlfoabjr",
  "info": {
    "created": "2023-10-17T14:43:26.181Z",
    "lastConnected": "2023-10-17T14:43:26.500Z"
  },
  "healthy": true,
  "selfHosted": true,
  "isConnectionActive": true,
  "description": "STH/OpenMetal instance created by Heat",
  "tags": [
    "openmetal",
    "medium"
  ],
  "topics": [],
  "sequences": [],
  "instances": []
}
```

#### 2. Change default Hub to be used:

```bash
si hub use sth-instance-sth_instance-2vd6hlfoabjr
```

#### 3. Check Hub version:

```bash
si hub version

{"service":"@scramjet/host","apiVersion":"v1","version":"0.36.1","build":"65b4dac"}
```

#### 4. Deploy sample sequence

Running a sample sequence will demonstrate that your hub is capable of receiving, executing, and managing sequences as expected.

```bash
# Download hello-world sequence
curl -s -O https://assets.scramjet.org/scp-store/hello-world-js.tar.gz

# Deploy the sequence directly to Self-Hosted Hub
si seq deploy hello-world-js.tar.gz| jq
{
  "_id": "4b16b343-914f-46b6-ba46-1cf2218d3007",
  "host": {
    "apiBase": "https://api.scramjet.cloud/api/v1/space/******/api/v1/sth/sth-instance-sth_instance-2vd6hlfoabjr/api/v1"
  },
  "sequenceURL": "sequence/4b16b343-914f-46b6-ba46-1cf2218d3007"
}

# Check running instances
si inst ls | jq

[
  {
    "id": "d31d146d-f759-4d77-aacb-747da2311216",
    "appConfig": {},
    "provides": "",
    "sequence": {
      "id": "4b16b343-914f-46b6-ba46-1cf2218d3007",
      "config": {
        "type": "docker",
        "container": {
          "image": "scramjetorg/runner:0.36.1",
          "maxMem": 512,
          "exposePortsRange": [
            30000,
            32767
          ],
          "hostIp": "0.0.0.0"
        },
        "name": "@scramjet/hello-world-js",
        "version": "0.0.1",
        "engines": {
          "node": ">=16"
        },
        "config": {},
        "sequenceDir": "/package",
        "entrypointPath": "index.js",
        "id": "4b16b343-914f-46b6-ba46-1cf2218d3007",
        "description": "Simple Sequence that prints out 'Hello World!' to the Instance output.",
        "author": "a-tylenda",
        "keywords": [
          "sample",
          "easy",
          "streaming",
          "Data Producer"
        ],
        "repository": {
          "type": "git",
          "url": "https://github.com/scramjetorg/platform-samples/tree/main/javascript/hello-world-js"
        },
        "language": "js"
      },
      "location": "sth-instance-sth_instance-2vd6hlfoabjr"
    },
    "created": "2023-10-18T17:38:08.479Z",
    "started": "2023-10-18T17:38:09.139Z",
    "status": "running"
  }
]

# Check instance output
si instance output d31d146d-f759-4d77-aacb-747da2311216 # Or shorter version: si inst output -
Hello World!
```

If the sample sequence deploys, executes, and provides the expected output without errors, it signifies that the Self-Hosted Hub is functioning properly and sequences can be managed as intended. If you encounter issues, refer to the Scramjet documentation (https://docs.scramjet.org) or support channels for assistance.

## Conclusion

Throughout this documentation, we've taken a comprehensive journey to set up and verify the Self-Hosted Hub on the OpenMetal platform. By following the provided steps, you've successfully:

- Understood the core capabilities of the Scramjet Transform Hub and its significance in data processing across diverse environments.
- Deployed a Heat Orchestration Template specifically designed for provisioning the Self-Hosted Hub.
- Confirmed the successful provisioning of the Self-Hosted Hub by deploying and running a sample sequence.

As a result, you now have a fully operational Scramjet Transform Hub running on the OpenMetal Cloud platform, ready to run data integration programs tailored to your specific needs. With this setup, the potential for scalable data processing and integration is at your fingertips.

For further information or any troubleshooting needs, always refer back to the Scramjet documentation or reach out to our dedicated support team. We hope this guide has empowered you to maximize the benefits of the Scramjet platform. Happy processing!

Read more: https://docs.scramjet.org
