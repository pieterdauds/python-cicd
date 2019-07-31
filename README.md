### Create New Project ###
$ oc new-project production
$ oc new-project testing
$ oc new-project development

### Inter-project permissions ###
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:production
role "system:image-puller" added: "system:serviceaccounts:production"
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:testing
role "system:image-puller" added: "system:serviceaccounts:testing"

### Populating Development ###
$ oc new-app https://github.com/python-cicd/hellopythonapp.git
$ oc expose svc hellopythonapp
route "hellopythonapp" exposed
$ curl $(oc get route hellopythonapp --template '{{.spec.host}}')
Hello Python World!

### Populating Testing & Production ###
$ oc project testing
$ oc tag development/hellopythonapp:latest hellopythonapp:test
$ oc new-app --image-stream=hellopythonapp:test
$ oc expose svc/hellopythonapp
route "hellopythonapp" exposed
$ curl $(oc get route hellopythonapp --template '{{.spec.host}}')
Hello Python World!

### Designing The Pipeline ###
node{
  stage('build & deploy') {
    openshiftBuild bldCfg: 'hellopythonapp',
    namespace: 'i3development',
      showBuildLogs: 'true'
    openshiftVerifyDeployment depCfg: 'hellopythonapp',
      namespace: 'i3development'
  }

  stage('approval (test)') {
    input message: 'Approve for testing?',
      id: 'approval'
  }

  stage('deploy to test') {
    openshiftTag srcStream: 'hellopythonapp',
      namespace: 'i3development',
      srcTag: 'latest',
      destinationNamespace: 'i3testing',
      destStream: 'hellopythonapp',
      destTag: 'test'
    openshiftVerifyDeployment depCfg: 'hellopythonapp',
      namespace: 'i3testing'
  }

  stage('approval (i3production)') {
    input message: 'Approve for production?',
      id: 'approval'
  }

  stage('deploy to production') {
    openshiftTag srcStream: 'hellopythonapp',
      namespace: 'i3development',
      srcTag: 'latest',
      destinationNamespace: 'i3production',
      destStream: 'hellopythonapp',
      destTag: 'prod'
    openshiftVerifyDeployment depCfg: 'hellopythonapp',
      namespace: 'i3production'
  }
}

### Starting Jenkins ###
$ oc new-project cicd
$ oc -n development policy add-role-to-user edit system:serviceaccount:cicd:jenkins
role "edit" added: "system:serviceaccount:cicd:jenkins"
$ oc -n testing policy add-role-to-user edit system:serviceaccount:cicd:jenkins
role "edit" added: "system:serviceaccount:cicd:jenkins"
$ oc -n production policy add-role-to-user edit system:serviceaccount:cicd:jenkins
role "edit" added: "system:serviceaccount:cicd:jenkins"
$ oc new-app https://github.com/pieterdauds/python-cicd.git#pipeline
$ oc logs bc/hellopythonapp

### Starting a new pipeline builds ###
$ oc -n cicd start-build hellopythonapp
