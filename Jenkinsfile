#!groovy
import groovy.json.JsonOutput
import groovy.json.JsonSlurper

def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, yaml: """
spec:
      containers:
      - name: mvn
        image: maven:3.3.9-jdk-8
        command:
        - cat
        tty: true
        volumeMounts:
        - mountPath: /cache
          name: maven-cache
      volumes:
      - name: maven-cache
        hostPath:
          # directory location on host
          path: /tmp
          type: Directory
""",    
{
    node (label) {
      container ('mvn') {
        // env.JAVA_HOME="${tool 'Java SE DK 8u131'}"
        // env.PATH="${env.JAVA_HOME}/bin:${env.PATH}"

        server = Artifactory.server "artifactory"
        buildInfo = Artifactory.newBuildInfo()
        buildInfo.env.capture = true

        (ref, commit) = checkout()
        // pull request or feature branch
        if  (env.BRANCH_NAME != 'master' && env.BRANCH_NAME != 'main') {
            // test whether this is a regular branch build or a merged PR build
            if (!isPRMergeBuild()) {
                // branch build
                codeQl = false
                build()
                unitTest()
                preview()
            } else {
                // PR-build
                codeQl = true
                initCodeQL()
                build()
                analyzeAndUploadCodeQLResults(ref, commit)  
                unitTest()
                // sonar()
            }    
        }
        else {
            // master/main branch / production
            codeQl = true
            initCodeQL()
            build()
            analyzeAndUploadCodeQLResults(ref, commit)
            allTests()
            preProduction()
            manualPromotion()
            production()
        }
    }
 }
})

def installCodeQL() {
      sh 'cd /tmp && test -f /tmp/codeql-runner-linux || curl -O -L  https://github.com/github/codeql-action/releases/latest/download/codeql-runner-linux'
      sh 'chmod a+x /tmp/codeql-runner-linux'
}

def initCodeQL() {
      stage 'Init CodeQL'
      installCodeQL()
      withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'ghe_token')]) {
            sh "/tmp/codeql-runner-linux init --repository ${getRepoSlug()} --github-url https://octodemo.com --github-auth \$ghe_token --languages java,javascript --config-file .github/codeq-config.yml"
      }
}

def analyzeAndUploadCodeQLResults(ref, commit) {
      stage 'Analyze & Upload CodeQL results'
      withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'ghe_token')]) {
            sh "/tmp/codeql-runner-linux analyze --repository ${getRepoSlug()} --github-url https://octodemo.com --github-auth \$ghe_token  --ref ${ref} --commit  ${commit}"
      } 
}

def isPRMergeBuild() {
    return (env.BRANCH_NAME ==~ /^PR-\d+$/)
}

def getPRNumber() {
    def matcher = (env.BRANCH_NAME =~ /^PR-(?<PR>\d+)$/)
    assert matcher.matches()
    return matcher.group("PR")
}

// in order for CodeQL to publish its results for PRs, the special ref refs/pull<PR>/merge has to get retrieved while checking out
// this can be done by telling Jenkins to add the following ref spec: +refs/pull/*/merge:refs/remotes/@{remote}/refs/pull/*/merge
def checkout () {
    stage 'Checkout code'
    context="continuous-integration/jenkins/"
    context += isPRMergeBuild()?"pr-merge/checkout":"branch/checkout"
    def scmVars = checkout scm
    setBuildStatus ("${context}", 'Checking out completed', 'SUCCESS')
    if (isPRMergeBuild()) {
      prMergeRef = "refs/pull/${getPRNumber()}/merge"
      mergeCommit=sh(returnStdout: true, script: "git show-ref ${prMergeRef} | cut -f 1 -d' '")
      echo "Merge commit: ${mergeCommit}"
      return [prMergeRef, mergeCommit]
    } else {
      return ["refs/heads/${env.BRANCH_NAME}", scmVars.GIT_COMMIT]    
    }
}

def build () {
    stage 'Build'
    mvn 'clean install -DskipTests=true -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -B -V'
}

def unitTest() {
    stage 'Unit tests'
    mvn 'test -B -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true'
    
    if (currentBuild.result == "UNSTABLE") {
        sh "exit 1"
    }
}

def sonar() {
    stage('SonarQube analysis') {
    prNo = (env.BRANCH_NAME=~/^PR-(\d+)$/)[0][1] 
    echo "PR-No: ${prNo}"
    // requires SonarQube Scanner 2.8+
    mvn 'org.jacoco:jacoco-maven-plugin:prepare-agent install -Dmaven.test.failure.ignore=true -Pcoverage-per-test'
    withCredentials([[$class: 'StringBinding', credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN']]) {
        githubToken=env.GITHUB_TOKEN
        withSonarQubeEnv('SonarQube Octodemoapps') {
            mvn "-Dsonar.analysis.mode=preview -Dsonar.github.pullRequest=${prNo} -Dsonar.github.oauth=${githubToken} -Dsonar.github.repository=Hyrule/reading-time-app -Dsonar.github.endpoint=https://octodemo.com/api/v3/ org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar"
            mvn "org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar"
        }
    }
    
    context="sonarqube/qualitygate"
    setBuildStatus ("${context}", 'Checking Sonarqube quality gate', 'PENDING')
    sleep 5
    timeout(time: 1, unit: 'MINUTES') { // Just in case something goes wrong, pipeline will be killed after a timeout
        def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
        if (qg.status != 'OK') {
            setBuildStatus ("${context}", "Sonarqube quality gate fail: ${qg.status}", 'FAILURE')
            error "Pipeline aborted due to quality gate failure: ${qg.status}"
        } else {
            setBuildStatus ("${context}", "Sonarqube quality gate pass: ${qg.status}", 'SUCCESS')
        }    
    }
  }
}

def allTests() {
    stage 'All tests'
    // don't skip anything
    mvn 'test -B'
    step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
    if (currentBuild.result == "UNSTABLE") {
        // input "Unit tests are failing, proceed?"
        sh "exit 1"
    }
}

def preview() {
    herokuApp = "${env.HEROKU_PREVIEW}"
    deployToStage("preview", herokuApp)
    // mvn 'deploy -DskipTests=true'
}

def preProduction() {
    switchSnapshotBuildToRelease()
    herokuApp = "${env.HEROKU_PREPRODUCTION}"
    deployToStage("preproduction", herokuApp)
    buildAndPublishToArtifactory()
}

def manualPromotion() {
  stage 'Manual Promotion'
    // we need a first milestone step so that all jobs entering this stage are tracked an can be aborted if needed
    milestone 1
    // time out manual approval after ten minutes
    timeout(time: 10, unit: 'MINUTES') {
        input message: "Does Pre-Production look good?"
    }
    // this will kill any job which is still in the input step
    milestone 2
}

def production() {
    herokuApp = "${env.HEROKU_PRODUCTION}"
    step([$class: 'ArtifactArchiver', artifacts: '**/target/*.jar', fingerprint: true])
    deployToStage("production", herokuApp)
    def version = getCurrentHerokuReleaseVersion("${env.HEROKU_PRODUCTION}")
    def createdAt = getCurrentHerokuReleaseDate("${env.HEROKU_PRODUCTION}", version)
    echo "Release version: ${version}"
    createRelease(version, createdAt)
    
    stage ("Promote in Artifactory") {
        promoteBuildInArtifactory()
        // distributeBuildToBinTray()
    }
}

void createRelease(tagName, createdAt) {
    withCredentials([[$class: 'StringBinding', credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN']]) {
        def body = "**Created at:** ${createdAt}\n**Deployment job:** [${env.BUILD_NUMBER}](${env.BUILD_URL})\n**Environment:** [${env.HEROKU_PRODUCTION}](https://dashboard.heroku.com/apps/${env.HEROKU_PRODUCTION})"
        def payload = JsonOutput.toJson(["tag_name": "v${tagName}", "name": "${env.HEROKU_PRODUCTION} - v${tagName}", "body": "${body}"])
        def apiUrl = "https://octodemo.com/api/v3/repos/${getRepoSlug()}/releases"
        def response = sh(returnStdout: true, script: "curl -s -H \"Authorization: Token ${env.GITHUB_TOKEN}\" -H \"Accept: application/json\" -H \"Content-type: application/json\" -X POST -d '${payload}' ${apiUrl}").trim()
    }
}

def deployToStage(stageName, herokuApp) {
    stage name: "Deploy to ${stageName}", concurrency: 1
    id = createDeployment(getBranch(), "${stageName}", "Deploying branch to ${stageName}")
    echo "Deployment ID for ${stageName}: ${id}"
    if (id != null) {
        setDeploymentStatus(id, "pending", "https://${herokuApp}.herokuapp.com/", "Pending deployment to ${stageName}");
        herokuDeploy "${herokuApp}"
        setDeploymentStatus(id, "success", "https://${herokuApp}.herokuapp.com/", "Successfully deployed to ${stageName}");
    }
}

def herokuDeploy (herokuApp) {
    withCredentials([[$class: 'StringBinding', credentialsId: 'HEROKU_API_KEY', variable: 'HEROKU_API_KEY']]) {
        mvn "heroku:deploy -DskipTests=true -Dmaven.javadoc.skip=true -B -V -D heroku.appName=${herokuApp}"
    }
}

def switchSnapshotBuildToRelease() {
    def descriptor = Artifactory.mavenDescriptor()
    descriptor.version = '1.0.0'
    descriptor.pomFile = 'pom.xml'
    descriptor.transform()
}

def buildAndPublishToArtifactory() {       
        def rtMaven = Artifactory.newMavenBuild()
        // rtMaven.tool = null
         withEnv(["MAVEN_HOME=/usr/share/maven", "JAVA_HOME=/usr/lib/jvm/java-1.8-openjdk/jre"]) {
           rtMaven.deployer releaseRepo:'libs-release-local', snapshotRepo:'libs-snapshot-local', server: server
           rtMaven.resolver releaseRepo:'libs-release', snapshotRepo:'libs-snapshot', server: server
           rtMaven.run pom: 'pom.xml', goals: 'install', buildInfo: buildInfo
           server.publishBuildInfo buildInfo
        }
}

def promoteBuildInArtifactory() {
        def promotionConfig = [
            // Mandatory parameters
            'buildName'          : buildInfo.name,
            'buildNumber'        : buildInfo.number,
            'targetRepo'         : 'libs-prod-local',
 
            // Optional parameters
            'comment'            : 'deploying to production',
            'sourceRepo'         : 'libs-release-local',
            'status'             : 'Released',
            'includeDependencies': false,
            'copy'               : true,
            // 'failFast' is true by default.
            // Set it to false, if you don't want the promotion to abort upon receiving the first error.
            'failFast'           : true
        ]
 
        // Promote build
        server.promote promotionConfig
}

def distributeBuildToBinTray() {
        def distributionConfig = [
            // Mandatory parameters
            'buildName'             : buildInfo.name,
            'buildNumber'           : buildInfo.number,
            'targetRepo'            : 'reading-time-dist',  
            // Optional parameters
            //'publish'               : true, // Default: true. If true, artifacts are published when deployed to Bintray.
            'overrideExistingFiles' : true, // Default: false. If true, Artifactory overwrites builds already existing in the target path in Bintray.
            //'gpgPassphrase'         : 'passphrase', // If specified, Artifactory will GPG sign the build deployed to Bintray and apply the specified passphrase.
            //'async'                 : false, // Default: false. If true, the build will be distributed asynchronously. Errors and warnings may be viewed in the Artifactory log.
            //"sourceRepos"           : ["yum-local"], // An array of local repositories from which build artifacts should be collected.
            //'dryRun'                : false, // Default: false. If true, distribution is only simulated. No files are actually moved.
        ]
        server.distribute distributionConfig
}

def mvn(args) {
    withMaven(
        // mavenSettingsConfig: '0e94d6c3-b431-434f-a201-7d7cda7180cb'
        
        //mavenLocalRepo: '/tmp/m2'
        ) {
 
      // Run the maven build
          if (codeQl) {
            sh ". codeql-runner/codeql-env.sh && mvn $args -Dmaven.test.failure.ignore -Dmaven.repo.local=/cache"
          } else {
            sh "mvn $args -Dmaven.test.failure.ignore -Dmaven.repo.local=/cache"
          }
     }
}

def shareM2(file) {
    // Set up a shared Maven repo so we don't need to download all dependencies on every build.
    writeFile file: 'settings.xml',
    text: "<settings><localRepository>${file}</localRepository></settings>"
}

def getRepoSlug() {
    tokens = "${env.JOB_NAME}".tokenize('/')
    org = tokens[tokens.size()-3]
    repo = tokens[tokens.size()-2]
    return "${org}/${repo}"
}

def getBranch() {
    tokens = "${env.JOB_NAME}".tokenize('/')
    branch = tokens[tokens.size()-1]
    return "${branch}"
}

def createDeployment(ref, environment, description) {
    withCredentials([[$class: 'StringBinding', credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN']]) {
        def payload = JsonOutput.toJson(["ref": "${ref}", "description": "${description}", "environment": "${environment}", "required_contexts": []])
        def apiUrl = "https://octodemo.com/api/v3/repos/${getRepoSlug()}/deployments"
        def response = sh(returnStdout: true, script: "curl -s -H \"Authorization: Token ${env.GITHUB_TOKEN}\" -H \"Accept: application/json\" -H \"Content-type: application/json\" -X POST -d '${payload}' ${apiUrl}").trim()
        def jsonSlurper = new JsonSlurper()
        def data = jsonSlurper.parseText("${response}")
        return data.id
    }
}

void setDeploymentStatus(deploymentId, state, targetUrl, description) {
    withCredentials([[$class: 'StringBinding', credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN']]) {
        def payload = JsonOutput.toJson(["state": "${state}", "target_url": "${targetUrl}", "description": "${description}"])
        def apiUrl = "https://octodemo.com/api/v3/repos/${getRepoSlug()}/deployments/${deploymentId}/statuses"
        def response = sh(returnStdout: true, script: "curl -s -H \"Authorization: Token ${env.GITHUB_TOKEN}\" -H \"Accept: application/json\" -H \"Content-type: application/json\" -X POST -d '${payload}' ${apiUrl}").trim()
    }
}

void setBuildStatus(context, message, state) {
  step([
      $class: "GitHubCommitStatusSetter",
      contextSource: [$class: "ManuallyEnteredCommitContextSource", context: context],
      errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
      reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://octodemo.com/${getRepoSlug()}"],
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
  ]);
}

def getCurrentHerokuReleaseVersion(app) {
    withCredentials([[$class: 'StringBinding', credentialsId: 'HEROKU_API_KEY', variable: 'HEROKU_API_KEY']]) {
        def apiUrl = "https://api.heroku.com/apps/${app}/dynos"
        def response = sh(returnStdout: true, script: "curl -s  -H \"Authorization: Bearer ${env.HEROKU_API_KEY}\" -H \"Accept: application/vnd.heroku+json; version=3\" -X GET ${apiUrl}").trim()
        def jsonSlurper = new JsonSlurper()
        def data = jsonSlurper.parseText("${response}")
        return data[0].release.version
    }
}

def getCurrentHerokuReleaseDate(app, version) {
    withCredentials([[$class: 'StringBinding', credentialsId: 'HEROKU_API_KEY', variable: 'HEROKU_API_KEY']]) {
        def apiUrl = "https://api.heroku.com/apps/${app}/releases/${version}"
        def response = sh(returnStdout: true, script: "curl -s  -H \"Authorization: Bearer ${env.HEROKU_API_KEY}\" -H \"Accept: application/vnd.heroku+json; version=3\" -X GET ${apiUrl}").trim()
        def jsonSlurper = new JsonSlurper()
        def data = jsonSlurper.parseText("${response}")
        return data.created_at
    }
}