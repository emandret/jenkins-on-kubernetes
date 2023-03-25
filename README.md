# Jenkins master-slave infrastructure on Kubernetes

## Create the cluster

Run:

```console
$ vagrant up
$ ansible-playbook -i inventory.yml master-playbook.yml --tags install-kubeadm
$ ansible-playbook -i inventory.yml worker-playbook.yml
$ ansible-playbook -i inventory.yml master-playbook.yml --tags install-jenkins
$ scp -r vagrant@192.168.1.0:/home/vagrant/.kube $HOME
```

> By default, Ansible invokes the playbook with `--tags all` which includes both `install-kubeadm` and `install-jenkins` roles.

If for some reasons you face issues with Kubernetes, run the following command to reset the cluster:

```console
$ ansible-playbook -i inventory.yml kubeadm-reset.yml
```

## Storage for Jenkins

The Jenkins master stores all its state on disk, in the `$JENKINS_HOME` directory, e.g., `$JENKINS_HOME` is `/var/jenkins_home` in the official Jenkins image. In order to preserve the configuration, build logs, and artifacts if the Jenkins master Pod dies, we'll use a `local` PV which exposes a host-local storage device such as a disk, partition or directory on specific node.

## Kubernetes plugin for Jenkins

The connection with the Kubernetes cluster is made possible thanks to the Kubernetes plugin. In the section below we'll install and configure the plugin accordingly.

First, go to the Jenkins dashboard by choosing an IP belonging to a node inside the cluster and the associated port as specified under `nodePort` in the `jenkins-http` service. Using the Vagrant configuration, the dashboard is accessible at each VM's IP address, e.g., http://192.168.1.2:31000.

Add a new a Kubernetes credential by going to `Manage Jenkins > Manage Credentials` and under `Stores scoped to Jenkins` select `(global) > Add credentials` and fill in the following values:

| Field  | Value                                                                                                                                                       |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Kind   | Secret text                                                                                                                                                 |
| Scope  | System (Jenkins and nodes only)                                                                                                                             |
| Secret | `kubectl get secret $(kubectl get sa jenkins-master -n jenkins -o jsonpath='{.secrets[0].name}') -n jenkins -o jsonpath='{.data.token}' \| base64 --decode` |
| ID     | [intentionally left blank]                                                                                                                                  |

Then go to `Manage Jenkins > Manage Nodes and Clouds > Configure Clouds > Add a new cloud > Kubernetes > [choose a name]` and click on `Kubernetes Cloud details...` and fill in the following values:

| Field                             | Value                                                                                                                                                         |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Kubernetes URL                    | `kubectl config view -o jsonpath='{.clusters[0].cluster.server}'`                                                                                             |
| Kubernetes server certificate key | `kubectl get secret $(kubectl get sa jenkins-master -n jenkins -o jsonpath='{.secrets[0].name}') -n jenkins -o jsonpath='{.data.ca\.crt}' \| base64 --decode` |
| Credentials                       | [choose the credential configured earlier]                                                                                                                    |
| Kubernetes Namespace              | `jenkins`                                                                                                                                                     |
| Jenkins URL                       | `http://jenkins-http`                                                                                                                                         |
| Jenkins tunnel                    | `jenkins-jnlp:50000`                                                                                                                                          |

Instead of keeping track of the service's virtual IP, we can use the built-in DNS resolution available across the cluster. Every service is accessible at `<service>.<namespace>.svc.cluster.local` or thanks to the search list in `/etc/resolv.conf` for hostname lookup, simply the service name, e.g., `jenkins-jnlp`.

Finally click on `Pod Templates...` to create a new Pod template and click on `Pod Template details...` and fill in the following values:

| Field     | Value                                                     |
| --------- | --------------------------------------------------------- |
| Name      | `jenkins-slave`                                           |
| Namespace | `jenkins`                                                 |
| Labels    | `jenkins-slave`                                           |
| Usage     | Only build jobs with label expressions matching this node |

> To use this slave Pod as an agent, we need to add: `agent { label 'jenkins-slave' }` at the beginning of the pipeline definition.

Under `Containers` edit the `Container Template` with the following values:

| Field        | Value                     |
| ------------ | ------------------------- |
| Name         | `ansible-runner`          |
| Docker image | `emandret/ansible-alpine` |

Eventually, click on `Save` and `Apply` to finish the setup. The whole configuration should be stored as a PV so that it doesn't have to be recreated each time the Jenkins master Pod is terminated.

## Scheduling

Jenkins will schedule build processes on the Jenkins master by default, to prevent this, we can explicitly set the number of executor processes to zero.

> The number of executor processes is configured in `Manage Jenkins > Manage Nodes and Clouds > [select a node] > Configure > # of executors`

When we trigger a job from a pipeline, the Jenkins master Pod will look for an agent to execute the job. The agent can be a physical machine, but in our case, the Jenkins master Pod will spawn an agent slave Pod.

The Pod template section is used to configure the slave Pod. By default, the template defines a single container named `jnlp` using the image `jenkins/agent` to establish an inbound connection to the Jenkins master using TCP or WebSockets.

> The Jenkins master Pod interacts directly with the Kubernetes API to execute commands inside containers, it doesn't preconfigure the agent to execute anything, see this [source code](https://github.com/jenkinsci/kubernetes-plugin/blob/master/src/main/java/org/csanchez/jenkins/plugins/kubernetes/pipeline/ContainerExecDecorator.java).

It isn't recommended to override the `jnlp` container, instead, we should add our own container definition to the template: we added a container named `ansible-runner` in the previous section.

The Jenkins master Pod needs to schedule the executor processes inside the containers so they shouldn't instantly die after having been created. For that, the container will idle with `sleep 9999` while waiting for a task.

> Don't override the default command: if the container can't idle, it will die before being able to execute any task.

Nodes can be defined in a pipeline and then used, however, default execution always goes to the `jnlp` container. You'll need to specify the name of the container you want to execute your task in.

This will run in the `jnlp` container:

```groovy
pipeline {
  agent {
    label 'jenkins-slave'
  }
  stages {
    stage('Test') {
      steps {
        sh "echo hello from $POD_CONTAINER"
      }
    }
  }
}
```

While this will run in the `ansible-runner` container:

```groovy
pipeline {
  agent {
    label 'jenkins-slave'
  }
  stages {
    stage('Test') {
      steps {
        container('ansible-runner') {
          sh "echo hello from $POD_CONTAINER"
        }
      }
    }
  }
}
```

## Sources

- https://devopscube.com/docker-containers-as-build-slaves-jenkins
- https://www.jenkins.io/doc/book/installing/kubernetes/#install-jenkins-with-yaml-files
- https://www.jenkins.io/doc/book/installing/kubernetes/#create-a-persistent-volume
