def TAG

podTemplate(
  serviceAccount: 'jenkins',
  containers: [
    containerTemplate(
      name: 'jnlp', 
      image: 'jenkins/inbound-agent:4.3-4',
      args: '${computer.jnlpmac} ${computer.name}'
    ),
    containerTemplate(
      name: 'kustomize',
      image: 'webfuel/kustomize:3.6.1',
      ttyEnabled: true,
      command: 'cat'
    )
  ],
  volumes: [
    hostPathVolume(mountPath: '/bin/kubectl', hostPath: '/bin/kubectl'),
  ]
)
{
    node(POD_LABEL) {
      stage('git scm update'){
          git url: 'https://github.com/IaC-Source/dashboard-deploy.git', branch: 'main'
      }      
      stage('define tag'){
          if(env.BUILD_NUMBER.toInteger() % 2 == 1){
              TAG = "blue"
          } else {
              TAG = "green"
          }
          echo "tag: $TAG"
      }
      stage('deploy configmap and deployment'){
        container('kustomize'){
              dir('deployment'){
                withEnv(["tag=${TAG}"]){
                sh '''
                  ls
                  echo "stage2-sh $tag"
                  kustomize create --resources ./deployment.yaml
                  echo "deploy new deployment"
                  kustomize edit add label deploy:$tag -f
                  kustomize edit set namesuffix -- -$tag
                  kustomize edit set image sysnet4admin/dashboard:$tag
                  kustomize build . | kubectl apply -f -
                  echo "retrieve new deployment"
                  kubectl get deployments -o wide
                '''
                }
            }
        }
      }
      stage('switching LB'){
        container('kustomize'){
          dir('service'){
            withEnv(["tag=${TAG}"]){
              sh '''
                kustomize create --resources ./lb.yaml
                while true;
                do
                export replicas=$(kubectl get deployments --selector=app=dashboard,deploy=$tag -o jsonpath --template="{.items[0].status.replicas}")
                export ready=$(kubectl get deployments --selector=app=dashboard,deploy=$tag -o jsonpath --template="{.items[0].status.readyReplicas}")
                echo "total replicas: $replicas, ready replicas: $ready"
                if [ "$ready" == "$replicas" ]; then
                  echo "Since all replicas have been deployed replace the target deployemnt of the loadbalancer"
                  kustomize edit add label deploy:$tag -f
                  kustomize build . | kubectl apply -f -
                  echo "delete old deployment resources"
                  kubectl delete deployment --selector=app=dashboard,deploy!=$tag
                  kubectl get deployments -o wide
                  break
                else
                  sleep 1
                fi
                done
              ''' 
            }
          }            
        }    
      }
    }
}
