# YCIT020 Falco demo
## 1. Set up a GKE cluster on Google Cloud
`gcloud container clusters create falco-demo --enable-network-policy --machine-type=e2-standard-4 --num-nodes=1 --release-channel=stable`

## 2. Add Falco Security official Helm repository
`helm repo add falcosecurity https://falcosecurity.github.io/charts`
 
`helm repo update`

## 3. Install Falco from Helm chart with default settings and eBPF
`helm install falco falcosecurity/falco --set ebpf.enabled=true`
 
*Note: You will get an error if you don't enable Falco in eBPF mode on GKE.*


## 4. Demo with Falco default rules and basic container
#### Start a simple pod with a container image that has lots of commands built-in (so we're not restricted in what we can run):
 
`kubectl run alpine -i --tty --attach=false --image=alpine`

#### In a separate terminal window, watch the log output of the Falco pod:
 
`kubectl logs -f <FALCO POD NAME>`

#### While watching the log output from the Falco container, exec into the simple container we started above:

`kubectl exec -i --tty alpine -- /bin/sh`

*Note that Falco alerts because a shell was spawned inside the container.*

#### In the container shell, try running a package manager command to install the SSH client (in this scenario, our hacker wants to look around laterally in the environment):

` apk add openssh`

*Note that Falco alerts because a known package manager is being run inside a container - containers should be immutable so it's not normal to have package managers installing new software.*

#### In the container shell, simulate someone trying to access the Kubernetes API from inside a container:

`wget 10.3.240.1:443`

*Note that Falco alerts because some unexpected access to the Kubernetes API could signal someone trying to discover or infiltrate the environment.*

## 5. Add a custom rule to alert for SSH usage inside the container
In my private GitHub repo (https://github.com/dpreno-mcgill/falco-demo/) I've pulled a copy of the Falco Helm chart, using:
 
`helm pull falcosecurity/falco --untar`
 
I've modified Falco's **falco_rules.yaml** file, which is where Falco reads the ruleset from on startup.
 
#### Tell Helm to update my running instance of 'falco' to include the data in my custom chart (pointing at the local 'falco' folder):
`cd ~/falco-demo/falco-chart`
 
`git pull`
 
`helm upgrade falco falco`
 
The local Helm chart _falco_ contains a modified falco_rules.yaml with a new rule called "Launch SSH client inside container".

#### Try running the SSH client in our container now:
`kubectl exec -i --tty alpine -- /bin/sh`
 
`ssh -V`
 
*Note that Falco sends my custom alert for SSH usage now.*

## 6. Add Slack integration to share alerts with your team when rules are triggered
#### We need to do 2 things in order to activate integration with Slack:
* **Create a custom Slack app** and generate an Incoming Webhook so that whoever has the webhook URL can post messages to channels - I followed [this procedure](https://api.slack.com/messaging/webhooks)
* Install **Falco Sidekick**, which can receive events from Falco and has built-in capabilities to interface with many external tools (including Slack)

The Slack preparation is well-documented in the link above, and not very interesting - so I've completed it ahead of time.
 
To implement the Falco changes, I'm going to take advantage of Helm values on the command line. The Falco chart is capable of pulling in Falco Sidekick and configuring it, all in one command.
 
#### Modify Falco to install Falco Sidekick and configure it, including my super-secret Slack Webhook URL:
 
`helm upgrade falco falco --set falco.httpOutput.enabled=true --set falco.httpOutput.url="http://falcosidekick:2801" --set falco.jsonOutput=true --set ebpf.enabled=true --set falcosidekick.enabled=true --set falcosidekick.webui.enabled=true --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/services/MY_SECRET_WEBHOOK_URL"`

#### Generate an alert again, see it hit the Slack instance
`kubectl exec -i --tty alpine -- /bin/sh`
  
`ssh -V`
 
*Note that Falco sends my custom alert for SSH to Slack now.*
 
## 7. Check out Falco SidekickUI
 
Forward a port from my local PC to get into the cluster service (no ingress configured):
 
`kubectl port-forward svc/falco-falcosidekick-ui 2802:2802`
 
Visit: http://127.0.0.1:2802/ui
