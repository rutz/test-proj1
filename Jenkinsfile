#!groovy

// variables used by script
isRelease = env.BRANCH_NAME == "master"
isPR = (env.JOB_BASE_NAME ==~ /PR-\d+/)
echo "isPR=${isPR}"

properties([disableConcurrentBuilds()])

node {
    try {
        sh 'env'
            stage('Checkout') {
                checkout scm
                env.MERGE_COMMIT = sh([
                    script      : "git rev-parse HEAD",
                    returnStdout: true
                ]).trim()
                echo "Git Commit: ${env.MERGE_COMMIT}"
                sh "git config --local remote.origin.url"
                env.GIT_REPO_NAME = sh([
                    script      : "git config --local remote.origin.url|sed -n 's#.*/\\([^.]*\\)\\.git#\\1#p'",
                    returnStdout: true
                ]).trim()

                env.GIT_REPO_USER = sh([
                    script      : "git config --local remote.origin.url|sed -n 's#.*/\\([^.]*\\)/.*\\.git#\\1#p'",
                    returnStdout: true
                ]).trim()
                env.CURRENT_GIT_AUTHOR = sh([
                    script      : "git --no-pager show -s --format=\"%an <%ae>\"",
                    returnStdout: true
                ]).trim()
                echo "Git Author: ${env.CURRENT_GIT_AUTHOR}"
                if (env.CURRENT_GIT_AUTHOR == 'Jenkins <jenkins@opendns.com>') {
                    currentBuild.result = "SUCCESS"
                    return
                }

                def remote = sh([script: "git ls-remote --get-url origin", returnStdout: true]).trim()
                print remote
                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'github-cred',
                                  usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                  sh "echo 'machine github.com login ${USERNAME} password ${PASSWORD}' >> ~/.netrc"
                  sh "chmod 600 ~/.netrc"
                
                    sh "git remote remove origin"
                    sh "git remote add origin ${remote}"
                    sh 'git fetch --prune origin \'+refs/tags/*:refs/tags/*\''
                }
            }
            if (isRelease) {
                sh "git log -1 --format='%h' > gitshash"
                def gitshash = readFile('gitshash').trim()
                stage('Docker Clean') {
                    sh 'docker ps -q -f status=exited | xargs --no-run-if-empty docker rm'
                    sh 'docker images -q -f dangling=true | xargs --no-run-if-empty docker rmi'
                    sh 'docker volume ls -qf dangling=true | xargs -r docker volume rm'
                }
                stage('Docker Build') {
                    sh "cat Dockerfile"
                    sh "docker build -t rutz/testproject:${gitshash} ."
                }
                stage('Docker Push') {
                    dir('.') {
                        sh "docker push rutz/testproject:${gitshash}"
                     }
                }
            }
        
    } catch (err) {
        throw err
    }
}