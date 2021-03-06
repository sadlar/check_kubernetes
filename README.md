# CHECK_KUBERNETES

Nagios-style checks against Kubernetes API. Designed for usage with Nagios, Icinga, Zabbix... Whatever.

## Dependencies

 * jq

## Script usage

    Usage ./check_kubernetes.sh [-m <MODE>|-h] [-o <TIMEOUT>] [-H <APISERVER> [-T <TOKEN>|-t <TOKENFILE>]] [-K <KUBE_CONFIG>]
             [-N <NAMESPACE>] [-n <NAME>] [-w <WARN>] [-c <CRIT>]
    
    Options are:
      -m MODE          Which check to perform
      -H APISERVER     API URL to query, kubectl is used if this option is not set
      -T TOKEN         Authorization token for API
      -t TOKENFILE     Path to file with token in it
      -K KUBE_CONFIG   Path to kube-config file for kubectl utility
      -N NAMESPACE     Optional namespace for some modes. By default all namespaces will be used
      -n NAME          Optional deployment name or pod app label depending on the mode being used. By default all objects will be checked
      -o TIMEOUT       Timeout in seconds; default is 15
      -w WARN          Warning threshold for pod restart count (in pods mode); default is 30
      -c CRIT          Critical threshold for pod restart count (in pods mode); default is 150
      -h               Show this help and exit
    
    Modes are:
      apiserver        Not for kubectl, should be used for each apiserver independently
      components       Check for health of k8s components (etcd, controller-manager, scheduler etc.)
      nodes            Check for active nodes
      pods             Check for restart count of containters in the pods
      deployments      Check for deployments availability
      daemonsets       Check for daemonsets readiness

## Examples:

Check apiserver health using tokenfile:

    ./check_kubernetes.sh -m apiserver -H https://<...>:6443 -t /path/to/tokenfile
    OK. Kuberenetes apiserver health is OK

Check whether all deployments are available using token:
    ./check_kubernetes.sh -m deployments -H https://<...>:6443 -T eYN6...
    OK. 27 deploymens are available

Check one definite deployment using kubectl:

    ./check_kubernetes.sh -m deployments -H https://<...>:6443 -K /path/to/kube_config -N ingress-nginx -n nginx-ingress-controller
    OK. Deployment available

Check nodes using kubectl with default kube config:

    ./check_kubernetes.sh -m nodes -H https://<...>:6443
    OK. 4 nodes are Ready

Check pods (by the restarts count):

    ./check_kubernetes.sh -m pods -H https://<...>:6443 -N kube-system -w 5
    WARNING. Container kube-system/calico-node-5kc4n/calico-node: 6 restarts. 22 pods ready, 0 pods not ready

Check daemonstets (compare number of desired and number of ready pods)
    ./check_kubernetes.sh -m daemonsets -K ~/.kube/cluster -N monitoring
    OK. Daemonset monitoring/node-exporter 5/5 ready


## ServiceAccount and token

All the needed objects (ServiceAccount, ClusterRole, RoleBinding) can be created with this command:

    kubectl apply -f https://raw.githubusercontent.com/agapoff/check_kubernetes/master/account.yaml

Then in order to get the token just issue this command:

    kubectl -n monitoring get secret "$(kubectl -n monitoring get serviceaccount monitoring -o 'jsonpath={.secrets[0].name}')" -o "jsonpath={.data.token}" | openssl enc -d -base64 -A

## Example configuration for Icinga

Command:

    object CheckCommand "check-kubernetes" {
      import "plugin-check-command"
    
      command = [ PluginDir + "/check_kubernetes.sh" ]
    
      arguments = {
                    "-H" = "$kube_scheme$://$host.address$:$kube_port$"
                    "-m" = "$kube_mode$"
                    "-o" = "$kube_timeout$"
                    "-T" = "$kube_pass$"
                    "-t" = "$kube_tokenfile$"
                    "-K" = "$kube_config$"
                    "-N" = "$kube_ns$"
                    "-n" = "$kube_name$"
                    "-w" = "$kube_warn$"
                    "-c" = "$kube_crit$"
      }
    }
    
Services:
    
    apply Service "k8s apiserver health" {
      import "generic-service"
      check_command = "check-kubernetes"
      vars.kube_mode = "apiserver"
      assign where "k8s-master" in host.vars.roles
    }
    
    apply Service "k8s components health" {
      import "generic-service"
      check_command = "check-kubernetes"
      vars.kube_mode = "components"
      assign where "k8s-api" in host.vars.roles
    }
    
    apply Service "k8s nodes" {
      import "generic-service"
      check_command = "check-kubernetes"
      vars.kube_mode = "nodes"
      assign where "k8s-api" in host.vars.roles
    }
    
    apply Service "k8s deployments" {
      import "generic-service"
      check_command = "check-kubernetes"
      vars.kube_mode = "deployments"
      assign where "k8s-api" in host.vars.roles
    }

    apply Service "k8s daemonsets" {
      import "generic-service"
      check_command = "check-kubernetes"
      vars.kube_mode = "daemonsets"
      assign where "k8s-api" in host.vars.roles
    }
    
    apply Service "k8s pods" {
      import "generic-service"
      check_command = "check-kubernetes"
      vars.kube_mode = "pods"
      assign where "k8s-api" in host.vars.roles
    }
    
    apply Service "k8s ingress controller" {
      import "generic-service"
      check_command = "check-kubernetes"
      vars.kube_mode = "deployments"
      vars.kube_ns = "ingress-nginx"
      vars.kube_name = "nginx-ingress-controller"
      assign where "k8s-api" in host.vars.roles
    }
    
Host template:

    template Host "k8s-host" {
      import "generic-host"
    
      vars.kube_pass = "..."
      vars.kube_scheme = "https"
      vars.kube_port = 6443
    }
    
Hosts:
   
    # VIP address of the API     
    object Host "k8s-api" {
      import "k8s-host"
      address = "<...>"
      vars.roles = [ "k8s-api" ]
    }
    
    object Host "k8s-master1" {
      import "linux-host"
      import "k8s-host"
      address = "<...>"
      vars.roles = [ "k8s-master" ]
}


## Licence: GNU GPL v3
