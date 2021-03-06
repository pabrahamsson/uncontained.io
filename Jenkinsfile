#!/usr/bin/groovy

////
// This pipeline requires the following plugins:
// Kubernetes Plugin 0.10
////

String ocpApiServer = env.OCP_API_SERVER ? "${env.OCP_API_SERVER}" : "https://openshift.default.svc.cluster.local"

node('master') {

  env.NAMESPACE = readFile('/var/run/secrets/kubernetes.io/serviceaccount/namespace').trim()
  env.TOKEN = readFile('/var/run/secrets/kubernetes.io/serviceaccount/token').trim()
  env.OC_CMD = "oc --token=${env.TOKEN} --server=${ocpApiServer} --certificate-authority=/run/secrets/kubernetes.io/serviceaccount/ca.crt --namespace=${env.NAMESPACE}"

  env.APP_NAME = "${env.JOB_NAME}".replaceAll(/-?pipeline-?/, '').replaceAll(/-?${env.NAMESPACE}-?/, '')
  def projectBase = "${env.NAMESPACE}".replaceAll(/-dev/, '')
  env.STAGE1 = "${projectBase}-dev"
  env.STAGE2 = "${projectBase}-prod"

  sh"""
    oc version
    oc get is jenkins-slave-image-mgmt -o jsonpath='{ .status.dockerImageRepository }' > /tmp/jenkins-slave-image-mgmt.out;
    oc get secret prod-credentials -o jsonpath='{ .data.api }' | base64 --decode > /tmp/prod_api;
    oc get secret prod-credentials -o jsonpath='{ .data.registry }' | base64 --decode > /tmp/prod_registry
    oc get secret prod-credentials -o jsonpath='{ .data.token }' | base64 --decode > /tmp/prod_token
  """
  env.SKOPEO_SLAVE_IMAGE = readFile('/tmp/jenkins-slave-image-mgmt.out').trim()
  env.PROD_API= readFile('/tmp/prod_api').trim()
  env.PROD_REGISTRY = readFile('/tmp/prod_registry').trim()
  env.PROD_TOKEN = readFile('/tmp/prod_token').trim()

  stage('Build Image') {
    sh "oc start-build ${env.APP_NAME} --wait --follow"
  }

}

podTemplate(label: 'promotion-slave', cloud: 'openshift', containers: [
  containerTemplate(name: 'jenkins-slave-image-mgmt', image: "${env.SKOPEO_SLAVE_IMAGE}", ttyEnabled: true, command: 'cat'),
  containerTemplate(name: 'jnlp', image: 'jenkinsci/jnlp-slave:2.62-alpine', args: '${computer.jnlpmac} ${computer.name}')
]) {

  node('promotion-slave') {

    stage("Promote To ${env.STAGE2}") {

      container('jenkins-slave-image-mgmt') {
        sh """

        set +x
        imageRegistry=\$(${env.OC_CMD} get is ${env.APP_NAME} --template='{{ .status.dockerImageRepository }}' -n ${env.STAGE1} | cut -d/ -f1)

        strippedNamespace=\$(echo ${env.NAMESPACE} | cut -d/ -f1)

        echo "Promoting \${imageRegistry}/${env.STAGE1}/${env.APP_NAME} -> \${PROD_REGISTRY}/${env.STAGE2}/${env.APP_NAME}"
        skopeo --tls-verify=false copy --remove-signatures --src-creds openshift:${env.TOKEN} --dest-creds openshift:${env.PROD_TOKEN} docker://\${imageRegistry}/${env.STAGE1}/${env.APP_NAME} docker://${PROD_REGISTRY}/${env.STAGE2}/${env.APP_NAME}
        """
      }
    }
  }
}
println "Application ${env.APP_NAME} is now in Production!"
