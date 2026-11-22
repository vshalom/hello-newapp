// GitOps flow: build the image (docker-in-docker), push tags <BUILD_NUMBER> + latest,
// render the Helm chart, and commit the rendered manifests to
//   https://github.com/elevy99927-demos/gitops  ->  systems/<DEPARTMENT>/<ENV>/<user>.yaml
// Argo CD (ApplicationSet/systems-manifests.yaml) deploys that file.
//
// Build agent: privileged docker-in-docker. It runs on any Kubernetes that allows privileged pods
// (EKS EC2 nodes, Rancher RKE2/k3s, OpenShift with the privileged SCC, minikube/kind) - see README.md.
// It doesn't use the node's container runtime (containerd/CRI-O), so the runtime doesn't matter.

properties([
  parameters([
    choice(name: 'ENV', choices: ['dev', 'qa', 'prd'], description: 'Environment folder: systems/<DEPARTMENT>/<ENV>/'),
    string(name: 'DEPARTMENT', defaultValue: 'hr', description: 'Department folder: hr, finance, legal, digital, devops'),
    booleanParam(name: 'SERVICE_MONITOR', defaultValue: false, description: 'Render the ServiceMonitor (needs the Prometheus Operator CRDs in the cluster)'),
    choice(name: 'DIND_STORAGE_DRIVER', choices: ['overlay2', 'vfs'], description: 'dind storage driver: overlay2 (fast). Use vfs only if overlay2 fails on your nodes (slow, uses more disk)')
  ])
])

def user        = "vshalom"              // your name: Docker Hub user AND gitops file name
def appname     = "onboardflow"
def appimage    = "${user}/${appname}"      // docker.io/<user>/helloworld
def apptag      = "${env.BUILD_NUMBER}"
def envName     = params.ENV ?: 'dev'
def department  = params.DEPARTMENT ?: 'hr'
def serviceMon  = params.SERVICE_MONITOR ? 'true' : 'false'
def storageDrv  = params.DIND_STORAGE_DRIVER ?: 'overlay2'
def gitopsRepo  = "github.com/elevy99927-demos/gitops.git"
def gitopsFile  = "systems/${department}/${envName}/${user}.yaml"

podTemplate(containers: [
      containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent', ttyEnabled: true),
      containerTemplate(name: 'deployer', image: 'elevy99927/k8s-deployer:latest', command: 'cat', ttyEnabled: true),
      // docker-in-docker: this container runs dockerd; the sh steps below run the docker CLI in it
      containerTemplate(name: 'docker', image: 'docker:27-dind', privileged: true,
                        args: "--storage-driver=${storageDrv}",
                        envVars: [envVar(key: 'DOCKER_TLS_CERTDIR', value: '')],   // no TLS inside the pod
                        resourceRequestCpu: '500m', resourceRequestMemory: '512Mi',
                        resourceLimitMemory: '2Gi')
  ],
  volumes: [
      // docker's data on the node's real disk: overlay2 works there (not on top of the container's overlayfs)
      emptyDirVolume(mountPath: '/var/lib/docker', memory: false)
  ])  {
    // Credentials come from Jenkins (Manage Jenkins -> Credentials), not from Kubernetes secrets:
    //   vshalom-dockerhub  : Username with password - <user> + Docker Hub access token
    //   github-token : Username with password - elevy99927-demos + GitHub token (write access to gitops)
    node(POD_LABEL) {
        stage('checkout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        }

        stage('build') {
            container('docker') {
                // wait for dockerd (started by the dind container) to accept connections
                sh '''
                    for i in $(seq 1 30); do docker info >/dev/null 2>&1 && exit 0; sleep 2; done
                    echo "docker daemon did not start"; docker info; exit 1
                '''
                // log in BEFORE the build: base images are pulled as <user>, not anonymously
                // (all pods behind one NAT IP share Docker Hub's anonymous pull limit)
                withCredentials([usernamePassword(credentialsId: 'vshalom-dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_TOKEN')]) {
                    sh "echo \"\$DOCKER_TOKEN\" | docker login -u \"\$DOCKER_USER\" --password-stdin"
                }
                echo "Building ${appimage}:${apptag}"
                sh "docker build --network=host -t ${appimage}:${apptag} -t ${appimage}:latest \"${env.WORKSPACE}\""
            }
        }

        stage('push') {
            container('docker') {
                // TAGS: <BUILD_NUMBER> (immutable, deployed by GitOps) and latest
                sh """
                    set -e
                    docker push ${appimage}:${apptag}
                    docker push ${appimage}:latest
                    docker logout
                """
            }
        }

        stage('deploy') {
            container('deployer') {
                echo "Rendering chart into ${gitopsRepo} -> ${gitopsFile}"
                withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                sh """#!/bin/sh
                    set -e
                    command -v git >/dev/null || apk add --no-cache git

                    rm -rf gitops
                    git clone https://\${GIT_USER}:\${GIT_TOKEN}@${gitopsRepo} gitops
                    cd gitops
                    git config user.name  "Jenkins (${user})"
                    git config user.email "elevy99927-demos@users.noreply.github.com"

                    helm template ${user} "${env.WORKSPACE}/chart" \\
                        --set image.repository=${appimage} \\
                        --set image.tag=${apptag} \\
                        --set serviceMonitor.enabled=${serviceMon} \\
                        > ${gitopsFile}

                    git add ${gitopsFile}
                    if git diff --cached --quiet; then
                        echo "No changes in ${gitopsFile} - nothing to deploy"
                        exit 0
                    fi
                    git commit -m "Deploy ${appimage}:${apptag} to ${department}/${envName} (build ${env.BUILD_NUMBER})"

                    # the whole class pushes to the same repo - rebase and retry on conflict
                    for i in 1 2 3 4 5; do
                        if git push origin HEAD:main; then exit 0; fi
                        echo "push rejected, rebasing (attempt \$i)"
                        git pull --rebase origin main
                        sleep \$i
                    done
                    echo "push failed after 5 attempts"; exit 1
                """
                }
            }
        }

        stage('add-dashboard') {
            container('deployer') {
                sh """
                    echo " TBD: Add dashboard to grafana"
                """
            }
        }
    }
  }
