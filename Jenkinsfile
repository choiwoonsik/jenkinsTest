pipeline {
    // 스테이지 별로 다른 거
    agent any

    triggers {
        pollSCM('*/1 * * * *')
    }

    environment {
      AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId')
      AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey')
      AWS_DEFAULT_REGION = 'ap-northeast-2'
      HOME = '.' // Avoid npm root owned
    }

    stages {
        // 레포지토리를 다운로드 받음
        stage('Prepare') {
            agent any
            
            steps {
                echo 'Clonning Repository'

                git url: 'https://github.com/choiwoonsik/jenkinsTest',
                    branch: 'master',
                    credentialsId: 'gitForJenkins'
            }

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success {
                  echo 'Successfully Pulled Repository'
                }

                always {
                  echo "i tried..."
                }

                cleanup {
                  echo "after all other post condition"
                }
            }
        }
        
        // aws s3 에 파일을 올림
        stage('Deploy Frontend') {
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance profile 을 등록해야함.
            dir ('./website'){
                // 배포를 위한 핵심 로직이 들어가는 부분이다.
                sh '''
                aws s3 sync ./ s3://wsjenkinss3
                '''
            }
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {
                  echo 'Successfully Pulled Repository'

                  slackSend (channel: '#jenkinstest', color: '#00FF00', message: "Successfully Deploy fronted")
              }

              failure {
                  echo 'I failed :('

                  // mail  to: 'dnstlr2933@gmail.com',
                  //       subject: "Failed Pipelinee",
                  //       body: "Something is wrong with deploy frontend"

                  slackSend (channel: '#jenkinstest', color: '#FF0000', message: "Failed Pipelinee")
              }
          }
        }
        
        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능!
            agent {
              docker {
                image 'node:latest'
              }
            }
            
            steps {
              dir ('./server'){
                  sh '''
                  npm install&&
                  npm run lint
                  '''
              }
            }
        }
        
        stage('Test Backend') {
          agent {
            docker {
              image 'node:latest'
            }
          }
          steps {
            echo 'Test Backend'

            dir ('./server'){
                sh '''
                npm install
                npm run test
                '''
            }
          }
        }
        
        stage('Bulid Backend') {
          agent any
          steps {
            echo 'Build Backend'

            dir ('./server'){
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }

          post {
            failure {
              error 'This pipeline stops here...'
            }
          }
        }
        
        stage('Deploy Backend') {
          agent any

          steps {
            echo 'Build Backend'

            dir ('./server'){
              // docker rm -f $(docker ps -aq) 컨테이너가 돌고 있을 경우 제거
                sh '''
                docker rm -f $(docker ps -aq)
                docker run -p 80:80 -d server
                '''
            }
          }

          post {
            success {
              // mail  to: 'dnstlr2933@gmail.com',
              //       subject: "Deploy Success",
              //       body: "Successfully deployed!"
              slackSend (channel: '#jenkinstest', color: '#00FF00', message: "Deploy Success")
            }
          }
        }
    }
}
