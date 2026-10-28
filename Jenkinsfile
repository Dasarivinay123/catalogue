@Library('jenkins-shared-library') _

def configMap = [
    project: "roboshop",
    component: "catalogue"
]

if (env.BRANCH_NAME.equalsIgnoreCase('main')){
    echo " we will deal later !!!"
}
else {
    nodejsEKSPipeline(configMap)
}
