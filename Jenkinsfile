@Library('jenkins-pipeline-shared-libraries')_

changeAuthor = env.ghprbPullAuthorLogin ?: CHANGE_AUTHOR
changeBranch = env.ghprbSourceBranch ?: CHANGE_BRANCH
changeTarget = env.ghprbTargetBranch ?: CHANGE_TARGET

pipeline {
    agent {
        label 'kie-rhel7 && kie-mem16g'
    }
    tools {
        maven 'kie-maven-3.6.2'
        jdk 'kie-jdk11'
    }
    options {
        timeout(time: 120, unit: 'MINUTES')
    }
    environment {
        MAVEN_OPTS = '-Xms1024m -Xmx4g'
    }
    stages {
        stage('Initialize') {
            steps {
                script {
                    mailer.buildLogScriptPR()

                    checkoutRuntimesRepo()
                    checkoutRepo('optaplanner')
                    String quickstartsRepo = 'optaplanner-quickstarts'
                    dir(quickstartsRepo) {
                        // If the PR to OptaPlanner targets the 'master' branch, we assume the branch 'development' for quickstarts.
                        String quickstartsChangeTarget = changeTarget == 'master' ? 'development' : changeTarget
                        githubscm.checkoutIfExists(quickstartsRepo, changeAuthor, changeBranch, 'kiegroup', quickstartsChangeTarget, true)
                    }
                }
            }
        }
        stage('Build Kogito Runtimes skipping tests') {
            steps {
                mavenCleanInstall("kogito-runtimes", true, [])
            }
        }
        stage('Build OptaPlanner') {
            steps {
                mavenCleanInstall('optaplanner', false, ['run-code-coverage'], '-Dfull')
            }
        }
        stage('Analyze OptaPlanner by SonarCloud') {
            steps {
                withCredentials([string(credentialsId: 'SONARCLOUD_TOKEN', variable: 'SONARCLOUD_TOKEN')]) {
                    runMaven("validate", "optaplanner", true, ["sonarcloud-analysis"], "-e -nsu -Dsonar.projectKey=org.optaplanner:optaplanner")
                }
            }
        }
        stage('Build OptaPlanner Quickstarts') {
            steps {
                mavenCleanInstall("optaplanner-quickstarts", false, [])
            }
        }
    }
    post {
        always {
            sh '$WORKSPACE/trace.sh'
            junit '**/target/surefire-reports/**/*.xml, **/target/failsafe-reports/**/*.xml'
            cleanWs()
        }
        failure {
            script {
                mailer.sendEmail_failedPR()
            }
        }
        unstable {
            script {
                mailer.sendEmail_unstablePR()
            }
        }
        fixed {
            script {
                mailer.sendEmail_fixedPR()
            }
        }
    }
}

void checkoutRepo(String repo, String dirName=repo) {
    dir(dirName) {
        githubscm.checkoutIfExists(repo, changeAuthor, changeBranch, 'kiegroup', changeTarget, true)
    }
}

void checkoutRuntimesRepo() {
    String targetBranch = changeTarget
    String [] versionSplit = targetBranch.split("\\.")
    if(versionSplit.length == 3 
        && versionSplit[0].isNumber()
        && versionSplit[1].isNumber()
       && versionSplit[2] == 'x') {
        targetBranch = "${Integer.parseInt(versionSplit[0]) - 7}.${versionSplit[1]}.x"
    } else {
        echo "Cannot parse changeTarget as release branch so going further with current value: ${changeTarget}"
    }
    dir('kogito-runtimes') {
        githubscm.checkoutIfExists('kogito-runtimes', changeAuthor, changeBranch, 'kiegroup', targetBranch, true)
    }
}

void mavenCleanInstall(String directory, boolean skipTests = false, List profiles = [], String extraArgs = "") {
    runMaven("clean install", directory, skipTests, profiles, extraArgs)
}

void runMaven(String command, String directory, boolean skipTests = false, List profiles = [], String extraArgs = "") {
    mvnCmd = command
    if(profiles.size() > 0){
        mvnCmd += " -P${profiles.join(',')}"
    }
    if(extraArgs != ""){
        mvnCmd += " ${extraArgs}"
    }
    dir(directory) {
        maven.runMavenWithSubmarineSettings(mvnCmd, skipTests)
    }
}
