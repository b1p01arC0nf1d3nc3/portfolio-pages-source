+++
title = 'TheHive'
date = 2023-12-09T19:04:27-04:00
draft = false
+++

During a work term, I was given the chance to build out some open source tools that can be used to create a streamlined security operations center. It was a very valuable experience and helped me develop a better understanding of tooling available to organizations that may have resource limitations. 

Here are some of the configuration steps and code that was used in the setup of the Hive server, a self-contained testing server to explore open-source SOC options.

Base OS: CentOS 7. 
Requirements: Depending on the number of products you want to test at once, this could range from 8 - 32 GB.

Docker engine: simple install using yum: 
`sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

Once docker is set up, start the service and check the inital `hello world` container can run. If it does, then we can start building the template for the SOC from the hosted page on github.


### Portainer
Using Portainer will make managing the large number of containers and images simpler. The container for Portainer can be installed by following the steps in this link:

 [Install Portainer with Docker on Linux - 
Portainer Documentation](https://docs.portainer.io/start/install-ce/server/docker/linux) 

The basic steps are given below.

`docker volume  name_of_data_storage_for_portainer` 

The above command will create a docker volume for Portainer. We can then run the docker command to pull in the image and map the ports we will use to connect to the web interface for managing containers.

I used: 

`docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v name_of_data_storage_for_portainer:/data portainer/portainer-ce:2.9.3` 

though your command may look different depending on the versions and the ports you select.

For the web interface, Portainer will automatically create a self-signed certificate, but this can be changed both during and after installation, should this be a production environment.

---
### SOC Components
The following tools were used to create the SOC, providing a wide range of functionality for small teams of security analysts.
- [ ] TheHive - incident platform to coordinate incident response and case collection
- [ ] Cortex - analysis of events based indicators of compromise
- [ ] MISP - shared threat intelligence platform
- [ ] Shuffle - automation platform
- [ ] ThePhish - email analysis and triage
- [ ] Wazuh - endpoint monitoring
- [ ] Assembly_Line - file analysis
- [ ] Cuckoo_Sandbox - automated sandbox analysis environments

The initial install takes care of many of the tools and will provide a backend database for storing case information. The details and highlights of configurations can be found at the following link. The template used includes an automation platform as well as integrations with MISP for sharing information with other SOCs.
[Docker-Templates/docker/thehive4-cortex3-misp-shuffle at main · TheHive-Project/Docker-Templates · GitHub](https://github.com/TheHive-Project/Docker-Templates/tree/main/docker/thehive4-cortex3-misp-shuffle)

After cloning the above repo with 

`git clone https://github.com/TheHive-Project/Docker-Templates/tree/main/docker/thehive4-cortex3-misp-shuffle` 

you can also clone ThePhish: 

`git clone https://github.com/emalderson/ThePhish.git`  

If there are any later configuration issues, you can find the repository here: [ThePhish/docker at master · emalderson/ThePhish · GitHub](https://github.com/emalderson/ThePhish/tree/master/docker) 

The docker-compose.yaml file will then need to be customized to include the following information in order to add ThePhish to the template.

>thephish:
     image: emalderson/thephish:latest
     container_name: thephish
     restart: unless-stopped
     depends_on:
       - thehive
       - cortex
       - misp
     ports:
       - '0.0.0.0:8081:8080'
     volumes:
       - ./ThePhish/app/analyzers_level_conf.json:/root/thephish/analyzers_level_conf.json
       - ./ThePhish/app/configuration.json:/root/thephish/configuration.json
       - ./ThePhish/app/whitelist.json:/root/thephish/whitelist.json


I added this as the last container in the sequence and there were no issues pulling the image. As there are a multitude of containers that will be hosting web applications, the port mappings will need to be adjusted to ensure there is no unecessary overlap, which is where Portainer helps out by making it simpler to identify these kinds of issues and jump between container logs.

The default port mappings can be found in the initial docker-compose.yaml file, though I needed to change the mappings for MISP as port 8080 and 443 were occupied (there were also issues with Shuffle seemingly accessing port 443, which caused MISP to not be contactable); the Shuffle frontend as common ports were used; and ThePhish for the same reason.

After all ports have been mapped, you can choose to select `0.0.0.0` or your static ip as the route. I found that adding the specific ip sometimes help get around issues where the container frontend was unreachable.

After customizing the configurations, you can build the apps by changing to the parent directory (where the docker-compose.yaml is located if you are not already there) and running `docker compose up -d` (may need to 'sudo' depending on how you configured docker) to begin the pull and build of all of the mentioned containers. You can check the status of the containers in the Portainer UI (reachable at https://server_ip:port_number). If there are issues with the containers, you can focus on them individually or bring them all down with `docker compose down`. You can then change the configuration in the docker-compose.yaml to remove any conflicts. When I integrated ThePhish, I did have some issues with conflicting files/directories, but this was likely due to trying to run the container for ThePhish in a separate compose and the paths to the configurations being misaligned. It might therefore be necessary to check the volume paths to ensure they are pointing in the right direction.

In its current iteration, ThePhish is designed to rely on a gmail account as the staging area for malicious emails to be sent. In order to change this, the ability to drag/drop or select local files could be added to the flask code that forms the basis of the app. However, for this setup a gmail address was created and used to contact the server. The configuration steps can be found here: [ThePhish/docker at master · emalderson/ThePhish · GitHub](https://github.com/emalderson/ThePhish/tree/master/docker) After adding the gmail account details and app password found in account management to the `configuration.json` file in the app directory within the container, you can also add the URL and API of TheHive and the Cortex instance. The inbox of the account will then be able to pass .eml files to the analyzers and integrate with TheHive and Cortex by automatically accessing analyzers and using a case template to send a response email once the mailer responder is accurately configured in Cortex. Cortex will need the email address to use, the host, use of port 587, and the SMTP user and password configured.


### User creation in TheHive, Cortex, and MISP
Default user logins for each app can be found in the docker template readme on github.  

To create a user in TheHive, you will first need to create an organization to assign them to. After using the default account to sign in, you can create the first organization and users by clicking through the UI with options generally in the top banner. You can set their passwords and generate API tokens that can be placed into app configuration files if needed.

Similarly, for Cortex, you will need to create an organization and users in the same manner.
The API token for the app instance can then be placed into the `application.conf` file within TheHive's directory on the server (for me this was located within my home directory where I cloned the template repo). This is also where you will need to add details about the MISP instance, such as the API key and the IP address/port where the instance can be reached.

To integrate MISP, grab the API key after logging in with the default credentials and creating the first organization. After adding the info to the `application.conf` file the integrations between the three apps should be ready to go. You will be able to spend some time looking at analyzers to set up and responders you would like to use. The configuration of some analyzers is done automatically.

### Adding the Shuffle webhook
As there are a lot of apps included in this version of Shuffle, the container may have issues with memory and caching of the apps on the initial load. Pausing the container and giving it some time to process the cache seemed to work and allowed the container to become more stable. As outlined in the docker template documentation, the webhook uri will need to be put into the `application.conf` file specifying the organization in TheHive. Webhooks are also disabled by default and will need to be enabled following the instructions in the readme. Unfortunately, the link they use for how do this specifically is broken, but you can use the example listed as a framework for how to fill in your user details and find the listed file. 
>read -p 'Enter the URL of TheHive: ' thehive_url
  read -p 'Enter your login: ' thehive_user
  read -s -p 'Enter your password: ' thehive_password

`curl -XPUT -u$thehive_user:$thehive_password -H 'Content-type: application/json' $thehive_url/api/config/organisation/notification -d '
``{
  ``"value": [
    ``{``
      ``"delegate": false,
      ``"trigger": { "name": "AnyEvent"},
      ``"notifier": { "name": "webhook", "endpoint": "local" }
    ``}``
  ``]``
``}``


Once the webhooks are enabled you should be able to begin making workflows in the Shuffle frontend.

## Wazuh in docker
The next piece of the SOC is Wazuh for endpoint management. The steps can be found here: [Wazuh Docker deployment - Deployment on Docker · Wazuh documentation](https://documentation.wazuh.com/current/deployment-options/docker/wazuh-container.html)
The path I followed was single-node deployment. Once the repo has been cloned, change to the single node directory and run the command to generate certificates using a provided container `docker-compose -f generate-indexer-certs.yml run --rm generator`. You can then go to the directory with the docker-compose.yaml and check the configuration to see if there are any port assignment clashes. There might also be a clash within the `opensearch_dashboards.yaml` file  in the /single-node/config/wazuh_dashboard directory. You can also change the information for the user login and password within the yaml file before running the group of containers with `docker compose up -d`.  Once set up, you can begin to add agents to the network and begin collecting data related to your endpoints. The default configuration contains a lot of rules that collect data related to privilege escalation and you can add some responses to common attacks such as a brute force or sql injection attack. The granularity you can achieve with these rules seems quite high. Results can be filtered by rule_id among a host of other options to identify relevant data and to view the effectiveness of a given rule. Rules can be added in  `/var/ossec/etc/rules/`. There is a large repository on github that has collections of rules that could be used as the basis for a production configuration.
The current up to date list of rules can be found here: [wazuh/ruleset at master · wazuh/wazuh · GitHub](https://github.com/wazuh/wazuh/tree/master/ruleset)


## Assembly Line
The tools to run Assembly Line in docker can be found here: [Docker - Assemblyline 4 (cybercentrecanada.github.io)](https://cybercentrecanada.github.io/assemblyline4_docs/installation/appliance/docker/). The setup is similar to the other containerized tools and requires a python environment, docker compose, and a firewall configuration for the minimal install.  After these changes, clone the repo with git clone: 
`git clone https://github.com/CybercentreCanada/assemblyline-docker-compose.git`and change into the `deployments/assemblyline` directory to begin making changes to the 
config.yaml file. It is set to run as a single node by default and has default login credentials that can be changed within the .env file inside of the deployment directory.

The next step is to create your https certificates or assign them from your local CA. If creating your own, this can be done with `openssl req -nodes -x509 -newkey rsa:4096 -keyout ~/deployments/assemblyline/config/nginx.key -out ~/deployments/assemblyline/config/nginx.crt -days 365 -subj "/C=CA/ST=Ontario/L=Ottawa/O=CCCS/CN=assemblyline.local"`.

When inside of the `deployments/assemblyline` directory, you can compose the build and the web interface will come up, reachable at the chosen port. After logging in, you can then configure which services you want to use from the ones available. The defaults have some utility and can break down .exe files and weblinks among other file types. Many of the tools do not require added configuraton, but the most useful will require some added API information and port connections (e.g., Yara). In order to get the most out of Cuckoo, for example, there is a decent amount of configuration required and the server you are using will have fairly high resource requirements.

## Cuckoo Sandbox
In order to create an environment that could be used to track the effects of a file/attachment in operation, you can setup up a Cuckoo VM which will allow you to then pass information to Assembly Line. This also allows you to monitor attempted network activity to some extent and to test the file within different OS environments. On the linux server I was using, this starts with creating a VM using KVM and assigning it an ip within an isolated network.

The configuration for the VM can be automated with a tool called VMCloak (documentation here: [Welcome to VMCloak’s documentation! — VMCloak 4.x documentation](https://vmcloak.readthedocs.io/en/latest/)) to allow you to quickly spin up new environments for testing. I did not use this approach as I was relying on a single server and the configuration likely would have required more resources than I had available. Using VMCloak would be a better production solution as it would streamline a lot of the configurations needed, and once 2/3 machine configurations that adequately duplicate the operating environments you want to focus on are made, they can be used within both Cuckoo and Assembly Line more easily.