podTemplate(
  serviceAccount: 'jenkins',
  containers: [
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
      stage('define tag'){
          sh '''
              echo "----- define tag"
              if [ $(($BUILD_NUMBER % 2)) -eq 1 ]
              then
                export tag="blue"
              else
                export tag="green"
              fi             
              '''
      }
      stage('deploy configmap and deployment'){
        container('kustomize'){
              dir('deployment'){
                sh '''
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
      stage('switching LB'){
        dir('service'){
                sh '''
                  kustomize create --resources ./lb.yaml
                  while true;
                  do
                  export replicas=$(kubectl get deployments --selector=app=echo-buildtime,deploy=$tag -o jsonpath --template="{.items[0].status.replicas}")
                  export ready=$(kubectl get deployments --selector=app=echo-buildtime,deploy=$tag -o jsonpath --template="{.items[0].status.readyReplicas}")
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
