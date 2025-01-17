#!/usr/bin/groovy

// Load helm shared library
@Library("github.com/camptocamp/jenkins-lib-helm")
def helm = new com.camptocamp.Helm()

podTemplate(name: 'geomapfish-builder', label: 'geomapfish', cloud: 'openshift', containers: [
    containerTemplate(
        name: 'jnlp',
        image: '172.30.26.108:5000/geomapfish-cicd/jenkins-slave-geomapfish:latest',
        ttyEnabled: true,
        command: '',
        privileged: false,
        alwaysPullImage: false,
        workingDir: '/tmp',
        args: '${computer.jnlpmac} ${computer.name}'
    )
  ]
){
  node('geomapfish'){
    withCredentials([usernamePassword(credentialsId: 'openshift-token-pw', usernameVariable: 'HELM_USER', passwordVariable: 'HELM_TOKEN')]) {
      def scmVars = checkout scm
      def params = readJSON file: 'Jenkinsfile.json'
      def package_params = readJSON file: 'package.json'

      // set additional git envvars for image tagging
      helm.gitEnvVars()

      // set chart version defined in package.json
      sh("sed -i.bak 's#version: 0.0.1#version: ${package_params.version}#' charts/${params.app.name}/Chart.yaml")

      // tag image with version, and branch-commit_id
      def image_tags_map = helm.getContainerTags()

      // compile tag list
      def image_tags_list = helm.getMapValues(image_tags_map)
      def dev_image_tag = "ref-${image_tags_list.get(0)}"
      def prod_image_tag = "v${package_params.version}"

      def pwd = pwd()
      def chart_dir = "${pwd}/${params.app.chart_dir}"
      def values_file = "${pwd}/Jenkinsfile.chart.values.yaml"

      def helm_chart = params.app.name
      def openshift_subdomain = params.openshift.domain

      // define release and namespaces (Caution: release must be unique over namespaces)
      // for testing
      def namespace_testing = params.openshift.namespace_testing
      def helm_release_testing = params.helm.release_testing

      // for dev
      def tiller_namespace_dev = params.openshift.tiller_namespace_dev
      def namespace_dev = params.openshift.namespace_dev
      def helm_release_dev = dev_image_tag
      def helm_release_last_dev = params.helm.release_last_dev

      // for staging
      def namespace_staging = params.openshift.namespace_staging
      def helm_release_staging = params.helm.release_staging

      // for prod
      def namespace_prod = params.openshift.namespace_prod
      def helm_release_prod = params.helm.release_prod

      // pipeline execution flags
      def debug = params.pipeline.debug
      def skip_build = params.pipeline.skip_build
      def skip_deploy = params.pipeline.skip_deploy
      def cleanup_ref_release = params.pipeline.cleanup_ref_release
      def deploy_last_dev_release = params.pipeline.deploy_last_dev_release
      def publish_helm_chart = params.pipeline.publish_helm_chart

      if (!skip_build) {
        stage('build-applications') {
            sh returnStdout: true, script: 'pwd'
            sh 'rm -rf node_modules || true'
            sh 'ln -s /usr/lib/node_modules .'
            sh returnStdout: true, script: 'make build'
        }

        stage('build-images') {
          openshift.withCluster() {
            openshift.doAs('openshift-token') {
              openshift.withProject( 'geomapfish-cicd' ){
                parallel (
                  "print" : {
                    echo "Active project: ${openshift.project()}"
                    echo """${
                      openshift.raw(
                        'start-build',
                        'demo-geomapfish-print',
                        '--from-dir',
                        './print',
                        '--wait',
                        '--follow'
                      ).out
                    }"""
                  },
                  "mapserver" : {
                    echo """${
                      openshift.raw(
                        'start-build',
                        'demo-geomapfish-mapserver',
                        '--from-dir',
                        './mapserver',
                        '--wait',
                        '--follow'
                      ).out
                    }"""
                  },
                  "wsgi" : {
                    echo """${
                      openshift.raw(
                        'start-build',
                        'demo-geomapfish-wsgi',
                        '--from-dir',
                        './',
                        '--wait',
                        '--follow'
                      ).out
                    }"""
                  }
                )    
              }
            }
          }
        }
      }

      stage('deploy-on-testing') {
        helm.login()

        if (debug) {
          echo "---------------- scm vars ----------------------------"
          for (var in scmVars) {
            println var
          }
          echo "---------------- env vars ----------------------------"
          sh 'env > env.txt'
          for (String i : readFile('env.txt').split("\r?\n")) {
            println i
          }
        }

        // tag the latest image commit sha
        openshiftTag(
          srcStream: 'demo-geomapfish-mapserver',
          srcTag: 'latest',
          destStream: 'demo-geomapfish-mapserver',
          destTag: dev_image_tag
        )
        openshiftTag(
          srcStream: 'demo-geomapfish-print',
          srcTag: 'latest',
          destStream: 'demo-geomapfish-print',
          destTag: dev_image_tag
        )
        openshiftTag(
          srcStream: 'demo-geomapfish-wsgi',
          srcTag: 'latest',
          destStream: 'demo-geomapfish-wsgi',
          destTag: dev_image_tag
        )

        helm.helmLint(chart_dir)

        // run dry-run helm chart installation
        helm.helmDeploy(
          dry_run       : true,
          name          : helm_release_testing,
          namespace     : namespace_testing,
          chart_dir     : chart_dir,
          values_file  : values_file,
          values : [
            "imageTag"            : dev_image_tag,
            "apps.wsgi.replicas"  : 1
          ]
        )

        // run helm chart installation
        helm.helmDeploy(
          name          : helm_release_testing,
          namespace     : namespace_testing,
          chart_dir     : chart_dir,
          values_file  : values_file,
          values : [
            "imageTag"            : dev_image_tag,
            "apps.wsgi.replicas"  : 1
          ]
        )

        helm.logout()

        openshiftVerifyDeployment(
          namespace: namespace_testing,
          depCfg: "$helm_release_testing-$helm_chart-mapserver"
        )
        openshiftVerifyDeployment(
          namespace: namespace_testing,
          depCfg: "$helm_release_testing-$helm_chart-print"
        )
        openshiftVerifyDeployment(
          namespace: namespace_testing,
          depCfg: "$helm_release_testing-$helm_chart-wsgi"
        )
      }

      stage('integration-test') {
        echo  'Wait 10 seconds for app to be ready'
        sleep 10
        sh "curl -f ${helm_release_testing}-${helm_chart}-wsgi-${namespace_testing}.${openshift_subdomain}/check_collector?"
        sh "curl -f ${helm_release_testing}-${helm_chart}-wsgi-${namespace_testing}.${openshift_subdomain}/check_collector?type=all"
      }

      // deploy only the master branch
      if (env.BRANCH == 'dev') {
        stage('deploy-on-dev') {
          helm.login()

          // cleanup testing env
          helm.helmDelete(
            name: helm_release_testing
          )

          // run dry-run helm chart installation
          helm.helmDeploy(
            dry_run           : true,
            name              : helm_release_dev,
            tiller_namespace  : tiller_namespace_dev,
            namespace         : namespace_dev,
            chart_dir         : chart_dir,
            values_file      : values_file,
            values : [
              "imageTag"            : dev_image_tag,
              "apps.wsgi.replicas"  : 1
            ] 
          )

          // run helm chart installation
          helm.helmDeploy(
            name              : helm_release_dev,
            tiller_namespace  : tiller_namespace_dev,
            namespace         : namespace_dev,
            chart_dir         : chart_dir,
            values_file      : values_file,
            values : [
              "imageTag"            : dev_image_tag,
              "apps.wsgi.replicas"  : 1
            ] 
          )
          if (cleanup_ref_release) {
          // cleanup testing env
            helm.helmDelete(
              tiller_namespace  : tiller_namespace_dev,
              name:               helm_release_dev
            )
          }
          if (deploy_last_dev_release) {
            // run helm chart installation of latest commit 
            helm.helmDeploy(
              name              : helm_release_last_dev,
              tiller_namespace  : tiller_namespace_dev,
              namespace         : namespace_dev,
              chart_dir         : chart_dir,
              values_file      : values_file,
              values : [
                "imageTag"            : dev_image_tag,
                "apps.wsgi.replicas"  : 1
              ] 
            )
          }
          helm.logout()
        }

      }

      // skip_deploy for testing env
      if (skip_deploy) {
        currentBuild.result = 'SUCCESS'
        return
      }

      // deploy only the master branch
      if (env.BRANCH == 'master') {
        stage('deploy-on-staging') {
          openshift.withProject( 'geomapfish-prod' ){
            // tag the latest image as staging
            openshiftTag(srcStream: 'demo-geomapfish-mapserver', srcTag: dev_image_tag, destStream: 'demo-geomapfish-mapserver', destTag: prod_image_tag)
            openshiftTag(srcStream: 'demo-geomapfish-print', srcTag: dev_image_tag, destStream: 'demo-geomapfish-print', destTag: prod_image_tag)
            openshiftTag(srcStream: 'demo-geomapfish-wsgi', srcTag: dev_image_tag, destStream: 'demo-geomapfish-wsgi', destTag: prod_image_tag)
          }

          helm.login()

          // cleanup testing env
          helm.helmDelete(
            name: helm_release_testing
          )

          // run dry-run helm chart installation
          helm.helmDeploy(
            dry_run       : true,
            name          : helm_release_staging,
            namespace     : namespace_staging,
            chart_dir     : chart_dir,
            values_file  : values_file,
            values : [
              "imageTag"            : prod_image_tag,
              "apps.wsgi.replicas"  : 2
            ] 
          )

          // run helm chart installation
          helm.helmDeploy(
            name          : helm_release_staging,
            namespace     : namespace_staging,
            chart_dir     : chart_dir,
            values_file  : values_file,
            values : [
              "imageTag"            : prod_image_tag,
              "apps.wsgi.replicas"  : 2
            ] 
          )
          helm.logout()
        }

        if (publish_helm_chart) {
          stage('publish-helm-chart') {
            podTemplate(name: 'skopeo', label: 'skopeo', cloud: 'openshift', containers: [
                containerTemplate(
                    name: 'jnlp',
                    image: '172.30.26.108:5000/geomapfish-cicd/jenkins-slave-skopeo:latest',
                    ttyEnabled: true,
                    command: '',
                    privileged: false,
                    alwaysPullImage: false,
                    workingDir: '/tmp',
                    args: '${computer.jnlpmac} ${computer.name}'
                )
              ]
            ){
              withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PWD')]) {
                node('skopeo'){
                  // mapserver
                  sh """skopeo copy \
                    --src-tls-verify=false \
                    --src-creds $HELM_USER:$HELM_TOKEN \
                    --dest-creds $DOCKERHUB_USER:$DOCKERHUB_PWD \
                    docker://172.30.26.108:5000/geomapfish-cicd/demo-geomapfish-mapserver:${prod_image_tag} \
                    docker://docker.io/camptocamp/demo-geomapfish-mapserver:${prod_image_tag}
                  """
                  // print
                  sh """skopeo copy \
                    --src-tls-verify=false \
                    --src-creds $HELM_USER:$HELM_TOKEN \
                    --dest-creds $DOCKERHUB_USER:$DOCKERHUB_PWD \
                    docker://172.30.26.108:5000/geomapfish-cicd/demo-geomapfish-print:${prod_image_tag} \
                    docker://docker.io/camptocamp/demo-geomapfish-print:${prod_image_tag}
                  """
                  // wsgi
                  sh """skopeo copy \
                    --src-tls-verify=false \
                    --src-creds $HELM_USER:$HELM_TOKEN \
                    --dest-creds $DOCKERHUB_USER:$DOCKERHUB_PWD \
                    docker://172.30.26.108:5000/geomapfish-cicd/demo-geomapfish-wsgi:${prod_image_tag} \
                    docker://docker.io/camptocamp/demo-geomapfish-wsgi:${prod_image_tag}
                  """

                  // also push latest tag on this images
                  // mapserver
                  sh """skopeo copy \
                    --src-tls-verify=false \
                    --src-creds $HELM_USER:$HELM_TOKEN \
                    --dest-creds $DOCKERHUB_USER:$DOCKERHUB_PWD \
                    docker://172.30.26.108:5000/geomapfish-cicd/demo-geomapfish-mapserver:${prod_image_tag} \
                    docker://docker.io/camptocamp/demo-geomapfish-mapserver:latest
                  """
                  // print
                  sh """skopeo copy \
                    --src-tls-verify=false \
                    --src-creds $HELM_USER:$HELM_TOKEN \
                    --dest-creds $DOCKERHUB_USER:$DOCKERHUB_PWD \
                    docker://172.30.26.108:5000/geomapfish-cicd/demo-geomapfish-print:${prod_image_tag} \
                    docker://docker.io/camptocamp/demo-geomapfish-print:latest
                  """
                  // wsgi
                  sh """skopeo copy \
                    --src-tls-verify=false \
                    --src-creds $HELM_USER:$HELM_TOKEN \
                    --dest-creds $DOCKERHUB_USER:$DOCKERHUB_PWD \
                    docker://172.30.26.108:5000/geomapfish-cicd/demo-geomapfish-wsgi:${prod_image_tag} \
                    docker://docker.io/camptocamp/demo-geomapfish-wsgi:latest
                  """
                }
              }
              dir('public-charts') {
                git credentialsId: 'git-charts', url: 'git@github.com:camptocamp/charts.git'
                sshagent(['git-charts']) {
                  sh """
                    helm package ../charts/${params.app.name} -d docs
                    helm repo index docs --url https://camptocamp.github.io/charts
                    git config --local user.email "jenkins@${params.openshift.domain}"
                    git config --local user.name "Jenkins"
                    git add .
                    git commit -m "update chart ${params.app.name} to version ${prod_image_tag}"
                    git push origin master
                  """
                }
              }
            }
          }
        }

        def Boolean promote
        promote = false
        stage('deploy-on-prod') {
          try {
            timeout(time: 7, unit: 'DAYS') {
              promote = input message: 'Input Required',
              parameters: [
                [ $class: 'BooleanParameterDefinition',
                  defaultValue: false,
                  description: "Check the box to deploy to Production",
                  name: "Deploy ${params.app.name} version ${prod_image_tag} to Production"
                ]
              ]
            }
          } catch (err) {
            // don't promote => no error
          }
          if (promote) {
            openshift.withProject( 'geomapfish-prod' ){
              // tag the latest image as staging
              openshiftTag(srcStream: 'demo-geomapfish-mapserver', srcTag: dev_image_tag, destStream: 'demo-geomapfish-mapserver', destTag: prod_image_tag)
              openshiftTag(srcStream: 'demo-geomapfish-print', srcTag: dev_image_tag, destStream: 'demo-geomapfish-print', destTag: prod_image_tag)
              openshiftTag(srcStream: 'demo-geomapfish-wsgi', srcTag: dev_image_tag, destStream: 'demo-geomapfish-wsgi', destTag: prod_image_tag)
            }
            helm.login()

            // run dry-run helm chart installation
            helm.helmDeploy(
              dry_run       : true,
              name          : helm_release_prod,
              namespace     : namespace_prod,
              chart_dir     : chart_dir,
              values_file  : values_file,
              values : [
                "imageTag"            : prod_image_tag,
                "apps.wsgi.replicas"  : 4
              ] 
            )

            // run helm chart installation
            helm.helmDeploy(
              name          : helm_release_prod,
              namespace     : namespace_prod,
              chart_dir     : chart_dir,
              values_file  : values_file,
              values : [
                "imageTag"            : prod_image_tag,
                "apps.wsgi.replicas"  : 4
              ] 
            )
            helm.logout()
          }
        }
      }
    }
  }
}