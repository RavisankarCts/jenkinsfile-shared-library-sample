@Library('jalogut/jenkins-basic-shared-library-sample') _
appname = "customer-consents"
wiremockappname ="wiremock-tip-customer-consents"
rabbitmqappname = "rabbitmq"

// JIRA
userstoryprefix="feature/"

// docker
registryhostPrd = "dtrpl.nl.corp.tele2.com:9443"
registryhost = "dtrdl.nl.corp.tele2.com:9443"
repositoryname = "development/fez-customer-consents"
swarmhost = "dle.nl.corp.tele2.com"
ucphost = "dockerdle.nl.corp.tele2.com"
ucpgroup = "/Shared/Development"
clientenvdle= "DOCKER_HOST=tcp://dockerdle.nl.corp.tele2.com:8443 DOCKER_TLS_VERIFY=1 DOCKER_CERT_PATH=/var/jenkins_home/client-bundles/dta-lan-client"

elasticurl = "elastic.tst.nl.corp.tele2.com:4560"

// hostnames
devhost = "${appname}.dev.${swarmhost}"
tsthost = "${appname}.tst.${swarmhost}"
inthost = "${appname}.int.${swarmhost}"
prfhost = "${appname}.prf.${swarmhost}"
uathost = "${appname}.uat.${swarmhost}"
wiremockhost = "${wiremockappname}.tst.${swarmhost}"
rabbitmqhost = "${appname}.${rabbitmqappname}.tst.${swarmhost}"

// stacknames
devstack = "${appname}_dev"
tststack = "${appname}_tst"
intstack = "${appname}_int"
prfstack = "${appname}_prf"
uatstack = "${appname}_uat"

//git data
stashkey = "FEZ"
stashrepo = "customer-consents"
giturl = "http://t2nl-devtooling.itservices.lan:7990/scm/fez/customer-consents.git"
giturlautotests = "http://t2nl-devtooling.itservices.lan:7990/scm/fez/autotests.git"
gitpullapiurl = "http://t2nl-devtooling.itservices.lan:7990/rest/api/1.0/projects/${stashkey}/repos/${stashrepo}/pull-requests"

// jenkins user id
userid = "nlsvc-jenkins-docker"

def buildDocker (version) {
    stage('Build Docker') {
    	dir("CustomerConsents"){
		    // Run the maven docker:build
			withCredentials([usernamePassword(credentialsId: "${userid}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
		        sh "docker login -u ${USERNAME} -p ${PASSWORD} ${registryhost}"
		    }
		    sh "docker build -t ${registryhost}/${repositoryname}:latest -t ${registryhost}/${repositoryname}:${version} . && docker push ${registryhost}/${repositoryname}:latest"
			sh "docker logout ${registryhost}"
		}
    }
}

def pushDocker(version, reghost, sign){
    withCredentials([usernamePassword(credentialsId: "${userid}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        sh "docker login -u ${USERNAME} -p ${PASSWORD} ${reghost}"
        if(sign){
            withCredentials([string(credentialsId: 'REPOSITORY_PASSPHRASE', variable: 'repo_phrase'), string(credentialsId: 'ROOT_PASSPHRASE', variable: 'root_phrase')]) {
                sh "DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE=${root_phrase} DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE=${repo_phrase} DOCKER_CONTENT_TRUST=1 docker push ${reghost}/${repositoryname}:${version}"
            }
        } else {
            sh "docker push ${reghost}/${repositoryname}:${version}"
        }
    }
    sh "docker logout ${reghost}"
}

def runAutotests(tag){
	dir('CustomerConsents/customer-consents-autotests'){
		try{
			sh "mvn verify -PcomponentTests -Dspring.profiles.active=tst -Dcucumber.options=\"--tags ${tag}\""
		} catch (err){
			error("An error occurred while running autotests " + err)
			throw err
		} finally {
            publishHTML (target: [
                    allowMissing: false,
                    alwaysLinkToLastBuild: false,
                    keepAll: true,
                    reportDir: 'target/site/cucumber-reports',
                    reportFiles: "feature-overview.html",
                    reportName: "AutoTests reports"
            ])
		}
	}
}

def runLoadtests(){
    dir('CustomerConsents/customer-consents-loadtests'){
        try{
            sh 'mvn gatling:test'
        } catch (err){
            error("An error occurred while running loadtests " + err)
            throw err
        } finally {
            sh "mv target/gatling/report* target/gatling/site"
            publishHTML (target: [
                    allowMissing: false,
                    alwaysLinkToLastBuild: false,
                    keepAll: true,
                    reportDir: 'target/gatling/site',
                    reportFiles: "index.html",
                    reportName: "LoadTest report"
            ])
        }
    }
}

def owasp(hostname){
	stage('Security Test') {
	    // Run zap scanner
	    sh "mkdir ${env.WORKSPACE}/owasp"
	    def containerid = sh returnStdout: true, script: "docker run --rm -v ${env.WORKSPACE}/owasp:/zap/wrk/:rw -t -d -u root dtrdl.nl.corp.tele2.com:9443/operations/zap2docker-stable zap-baseline.py -t http://${hostname}/customers/ -t http://${hostname}/resellers/ -t http://${hostname}/health -i -m5 -j -a -r zapreport.html"
	    retry = 0
	    while(!fileExists("${env.WORKSPACE}/owasp/zapreport.html")){
            if(retry > 20){
                error('took to long for the zap to make a perort to be deployed')
            }
            sleep 30
            retry++
	    }
	    // obtain the report
	    publishHTML (target: [
            allowMissing: false,
            alwaysLinkToLastBuild: false,
            keepAll: true,
            reportDir: 'owasp',
            reportFiles: "zapreport.html",
            reportName: "Security Report(OWASP)"
	    ])
    }
}

def createPullRequest(source_branch, dest_branch){
    stage('Create STASH Pull request') {
        def json = readFile "${env.WORKSPACE}/CustomerConsents/restmerge.json"
        json = json.replaceAll("#KEY#", stashkey)
        json = json.replaceAll("#REPOS#", stashrepo)
        json = json.replaceAll("#DESTBRANCH#", dest_branch)
        writeFile   file: "restmerge.json" , text: json.replaceAll("#BRANCH#", source_branch)
        withCredentials([usernamePassword(credentialsId: "${userid}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            def call = "curl -u ${USERNAME}:${PASSWORD} -X POST ${gitpullapiurl} -H 'Content-Type:application/json' -d  @restmerge.json"
            def result = sh(returnStdout:true, script: call)
            if(result.contains("errors") && !result.contains("DuplicatePullRequestException")){ //No error if pull request already exists
                error("Merge failed with error" + result)
                currentBuild.result = 'FAILURE'
            }
        }
    }
}

properties([buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '3', numToKeepStr: '5')), disableConcurrentBuilds(), pipelineTriggers([])])

node {
    try {
        deleteDir()
	    
	    stage('Run/Test Stack') {
                // set the version #VERSION#
                promote (tststack, tsthost, "latest", "tst")
                //runAutotests("@Component")
                //runLoadtests()
                //owasp(tsthost)
            }

        if (env.BRANCH_NAME == "master") {
            //buildDocker(pom.version)

            

        }

	} catch (error) {
        throw error
	}
}
