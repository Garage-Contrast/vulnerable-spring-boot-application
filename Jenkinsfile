pipeline {
  agent {
    kubernetes {
      idleMinutes 5
      yaml '''
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
    - name: contrast
      image: ghcr.io/garage-contrast/contrast-client-go:sha-b931941
      command: ["tail", "-f", "/dev/null"]      
    - name: openshift
      image: registry.redhat.io/openshift4/ose-jenkins-agent-base:latest
      command: ["tail", "-f", "/dev/null"]
      env:
        - name: LANG
          value: ""
        - name: LC_ALL
          value: ""
    - name: yq
      image: docker.io/mikefarah/yq:latest
      command: ["tail", "-f", "/dev/null"]
'''
    }
  }
  // environment {
  //   // APP_NAME = ''
  //   // DEPLOYMENT = ''
  //   // PROJECT = ''
  //   // CONTRAST_URL = ''
  //   // CONTRAST_APIKEY = ''
  //   // CONTRAST_SERVICEKEY = ''
  //   // CONTRAST_USERNAME = ''
  // }
  stages {
    stage('preamble') {
        steps {
            script {
                openshift.withCluster() {
                    openshift.withProject() {
                        echo "Using project: ${openshift.project()}"
                    }
                }
            }
        }
    }
    stage('configure build') {
      steps {
        sh '''
cat > buildconfig-template.yaml << 'EOF'
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: ${DEPLOYMENT}-intermed
spec:
  source:
    type: Git
    git:
      uri: $GIT_URL
  strategy:
    type: Docker
    dockerStrategy:
      dockerfilePath: Dockerfile
  output:
    to:
      kind: ImageStreamTag
      name: ${DEPLOYMENT}:intermediate
---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: $DEPLOYMENT
spec:
  lookupPolicy:
    local: true
---
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: ${DEPLOYMENT}-contrast
spec:
  source:
    dockerfile: |
      FROM please/replace/this:latest

      RUN mkdir -p /opt/contrast \
      && mvn dependency:copy -Dartifact=com.contrastsecurity:contrast-agent:LATEST -DoutputDirectory=/opt/contrast \
      && mv /opt/contrast/contrast-agent*.jar /opt/contrast/contrast-agent.jar
      
      ENV JAVA_TOOL_OPTIONS="-javaagent:/opt/contrast/contrast-agent.jar"
  strategy:
    type: Docker
    dockerStrategy:
      from:
        kind: ImageStreamTag
        name: ${DEPLOYMENT}:intermediate
  output:
    to:
      kind: ImageStreamTag
      name: ${DEPLOYMENT}:${GIT_COMMIT}
  triggers:
    - type: ImageChange
      imageChange: {}
EOF
'''
        container('openshift') {
          sh "DEPLOYMENT=${env.DEPLOYMENT} GIT_URL=${env.GIT_URL} GIT_COMMIT=${env.GIT_COMMIT} envsubst '\$DEPLOYMENT,\$GIT_COMMIT,\$GIT_URL' < \"buildconfig-template.yaml\" > \"buildconfig.yaml\""
        }
        container('openshift') {
          script {
            openshift.withCluster() {
              openshift.withProject() {
                def buildconfig = openshift.apply(readFile( './buildconfig.yaml' ))
              }
            }
          }
        }
      }
    }
    stage('build') {
      steps {
        container('openshift') {
          script {
            openshift.withCluster() {
              openshift.withProject() {
                def bc = openshift.selector('bc', "${env.DEPLOYMENT}-intermed")
                def buildSelector = bc.startBuild()
                buildSelector.logs('-f')

                def contrastBC = openshift.selector('bc', "${env.DEPLOYMENT}-contrast")
                def builds = contrastBC.related('builds')
                builds.untilEach(1) {
                  return it.object().status.phase == "Complete"
                }
              }
            }
          }
        }
      }
    }
    stage('Customize Deployment') {
      steps {
        container('openshift') {
          script {
            openshift.withCluster() {
              openshift.withProject() {
                openshift.apply(openshift.raw("create configmap contrast-cm-${env.DEPLOYMENT} --from-literal=CONTRAST__API__URL=${env.CONTRAST_URL} --from-literal=CONTRAST__AGENT__JAVA__STANDALONE_APP_NAME=${env.APP_NAME} --from-literal=CONTRAST__APPLICATION__NAME=${env.APP_NAME} --dry-run -o yaml").actions[0].out)
                openshift.apply(openshift.raw("create secret generic contrast-secret-${env.DEPLOYMENT} --from-literal=CONTRAST__API__API_KEY=${env.CONTRAST_APIKEY} --from-literal=CONTRAST__API__SERVICE_KEY=${env.CONTRAST_SERVICEKEY} --from-literal=CONTRAST__API__USER_NAME=${env.CONTRAST_USERNAME} --dry-run -o yaml").actions[0].out)
              }
            }
          }
        }
        container('yq') {
          sh '''
cd manifests
RESOURCES=$(ls)
echo 'resources:' | tee -a kustomization-template.yaml
for res in $RESOURCES ; 
do
  (yq ea -N -e '. | has("apiVersion")' $res  >> /dev/null && echo "- $res" | tee -a kustomization-template.yaml ) || echo "$res is invalid"
done

cat >> kustomization-template.yaml << 'EOF'
images:
- name: .*${DEPLOYMENT}.*
  newTag: $GIT_COMMIT
  newName: image-registry.openshift-image-registry.svc:5000/${NAMESPACE}/${DEPLOYMENT}
patchesStrategicMerge:
- |-
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: ${DEPLOYMENT}
  spec:
    template:
      spec:
        containers:
          - name: ${DEPLOYMENT}
            envFrom:
            - configMapRef:
                name: contrast-cm-${DEPLOYMENT}
            - secretRef:
                name: contrast-secret-${DEPLOYMENT}
EOF
'''
        }
        container('openshift') {
          sh "cd manifests && DEPLOYMENT=${env.DEPLOYMENT} NAMESPACE=${env.PROJECT} GIT_COMMIT=${env.GIT_COMMIT} envsubst '\$DEPLOYMENT,\$NAMESPACE,\$GIT_COMMIT' < \"kustomization-template.yaml\" > \"kustomization.yaml\""
        }
      }
    }
    stage('deploy') {
      steps {
        container('openshift') {
          script {
            openshift.withCluster() {
              openshift.withProject() {
                sh "ls -lah manifests/"
                sh "cat manifests/kustomization.yaml"
                sh "oc kustomize manifests"
                sh "oc kustomize manifests | oc apply -f -"
              }
            }
          }
        }
      }
    }
  }
}
