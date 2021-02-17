#!groovy​

// Example template for webMethods API Gateway Devops pipeline
// Developed by John Carter (john.carter@softewareag.com)
// November 2018

// Adapted 2021-02-17 preparation of the envs before demo

// Run command on remote server, requires propagation of jenkins server key to remote server
def ssh(id, server, command) {
	sh("ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no $id@$server $command")
}

// run ant command on remote server
def ant(id, server, command) {
	ssh(id, server, "ant $command")	
}

// format user password into authorization string
def authString(user, password) {

	 def encoded = "${user}:${password}".bytes.encodeBase64().toString()

	 return "Basic $encoded"
}

// check if Integration Server (API Gateway) is running on remote server for given port
def isISRunningInRemoteServer(server, port) {
	
	CONTAINER_RUNNING=true;
	
	try {
		sh "curl -s -o /dev/null -w '%{http_code}' $server:$port"
		
		IS_RUNNING=true
		echo "OK"
	} catch (exc) {

		IS_RUNNING=false
		echo "KO"
	}
	
	return IS_RUNNING
}

// deploy API from one API Gateway to another via Promotion
// Remote gateway is configure via Stage properties.
def promoteAPI(apigwUrl, stage, apis) {

	apiString = null;

	apis.each{ a -> 

		if (apiString == null) {
			apiString = "[\"${a}\""
		} else {		
			apiString = apiString + ",\"${a}\""
		}

		setAPIMaturity(apigwUrl, a, "UAT")
	}

	apiString = apiString + "]"

	def body = """ {
		"description": "Tested APIS ${apis}",
		"name": "CDI-${BUILD_NUMBER}",
		"destinationStages": ["${stage}"],
		"promotedAssets": {
    		"api": ${apiString}
  		}
	}"""

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/promotion", 
				validResponseCodes: '200:201'
}

// publish the API to the given API Portal and community
// Determine id's for stage, portal and communitiy via API
def publishAPI(apigwUrl, stage, id, portalName, communityName) {

	if (stage != null) {

		response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				url: "${apigwUrl}/rest/apigateway/stages/" + stage, 
				validResponseCodes: '200'


		jsn = readJSON file: '', text: "${response.content}"

		//def url = jsn.stages[0].url;
		//def name = jsn.stages[0].name;

		// publish from API Gateway indicated by staging (NOT main!)
		url = jsn.stage.url;
		auth = jsn.stage.name;

	} else {

		url = apigwUrl;
		auth = "wm-apigateway"
	}

	portalId = getPortalId(url, auth,  portalName)
	communityId = getPortalCommunityId(url, auth, portalId, communityName)

	def body = """ {
		"portalGatewayId": "${portalId}",
		"communities": ["${communityId}"],
		"endpoints": ["${apigwUrl}/rad/${id}"]
	}"""

	println("Publishing API's via "+url+" from "+auth+" to " + portalName)

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: auth, 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${url}/rest/apigateway/apis/" + id + "/publish", 
				validResponseCodes: '200'
}

// Removes given API from API Portal
def unpublishAPI(apigwUrl, apiName, id, portalId, communityId) {

	def body = """ {
		"portalGatewayId": "${portalId}",
		"communities": ["${communityId}"]
	}"""

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis/" + id + "/unpublish", 
				validResponseCodes: '200'

}

// Change maturity level for given API.
// maturity labels are set via extended property 'apiMaturityStatePossibleValues'

def setAPIMaturity(apigwUrl, id, maturity) {

   apiWrapper = getAPI(apigwUrl, id);

   println("Setting maturity to " + maturity + " for version " + apiWrapper.apiResponse.api.apiVersion);

    //pain, have to deactivate before update, really need option to only update attribute, NOT apiDefinition
    if (apiWrapper.apiResponse.api.isActive)
    	deactivateAPI(apigwUrl, id);
	
	// fixed in 10.4
	// bug in response, means the type is a string list, whereas it should be a string!!
	//apiWrapper.apiResponse.api.apiDefinition.type = apiWrapper.apiResponse.api.apiDefinition.type[0];

	raw = apiWrapper.apiResponse.api.apiDefinition.toString();

	def body = """{
		"maturityState": "${maturity}",
		"apiVersion": "${apiWrapper.apiResponse.api.apiVersion}",
  		"apiDefinition": ${raw}
	}"""

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis/${id}", 
				validResponseCodes: '200:402'

	activateAPI(apigwUrl, id);

	return response.content;
}

def getPortalId(apigwUrl, auth, portalName) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: auth, 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/portalGateways", 
				validResponseCodes: '200'

	def jsn = readJSON file: '', text: "${response.content}"

	def id = null;

	jsn.portalGatewayResponse.each { portal -> 
		
		if (portal.gatewayName == portalName) {

			id = portal.id;

			println("found id ${id} for portal name ${portalName}");
		}
	}
	
	return id;
}

def getPortalCommunityId(apigwUrl, auth, portalId, communityName) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: auth, 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/portalGateways/communities?portalGatewayId="+portalId, 
				validResponseCodes: '200'

	def jsn = readJSON file: '', text: "${response.content}"

	def id = null;

	jsn.portalGatewayResponse.communities.portalCommunities.each { c -> 
		
		if (c.name == communityName) {

			id = c.id;

			println("found id ${id} for community name ${communityName}");
		}
	}
	
	return id;
}

// Creation of Stage
def createStage(apigwUrl, stageName, stageDescription, stageURL, stageUsername, stagePwd) { // assuming default keystore and alis

	def body = """{
	   "name": "${stageName}",
	   "description": "${stageDescription}",
	   "url": "${stageURL}",
	   "username": "${stageUsername}",
	   "password": "${stagePwd}",
	   "keystoreAlias": "DEFAULT_IS_KEYSTORE",
	   "keyAlias": "ssos"
	}"""
	println("Body is : ${body} for stage name ${stageName}");

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/stages", 
				validResponseCodes: '200:299'

	def jsn = readJSON file: '', text: "${response.content}"

	def id = jsn.stage.id;
	
	return id;
}

// Creation of Alias
def createAlias(apigwUrl, stageID, aliasName, aliasDescription, aliasValue ) {

	def body = """{
	   "name": "${aliasName}",
	   "description": "${aliasDescription}",
	   "value": "${aliasValue}",
	   "stage": "${stageID}",
	   "type": "simple"
	}"""
	println("Body is : ${body} ");

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/alias", 
				validResponseCodes: '200:299'

}


def getStageId(apigwUrl, stageName) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/stages", 
				validResponseCodes: '200'

	def jsn = readJSON file: '', text: "${response.content}"

	def id = null;

	jsn.stages.each { s -> 
		
		if (s.name == stageName) {

			id = s.id;

			println("found id ${id} for stage name ${stageName}");
		}
	}
	
	return id;
}

def getApplicationId(apigwUrl, appName) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/applications", 
				validResponseCodes: '200'

	def jsn = readJSON file: '', text: "${response.content}"

	def id = null;

	jsn.applications.each { a -> 
		
		if (a.name == appName) {

			id = a.id;

			println("found id ${id} for application name ${appName}");
		}
	}
	
	if (id == null)
		println("No app found for ${appName}");

	return id;
}

// Fetch API details from API Gateway
def getAPI(apigwUrl, id) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/apis/" + id, 
				validResponseCodes: '200'

	//if (raw) {
	//	return response.content;
	//} else {
		def jsn = readJSON file: '', text: "${response.content}"

		return jsn;
	//}
}

// Activate API for use
def activateAPI(apigwUrl, id) {

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				url: "${apigwUrl}/rest/apigateway/apis/" + id + "/activate", 
				validResponseCodes: '200'

}

// Deactivate API on API gateway, will no longer be available, i.e. produce 404 error code
def deactivateAPI(apigwUrl, id) {

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				url: "${apigwUrl}/rest/apigateway/apis/" + id + "/deactivate", 
				validResponseCodes: '200'

}

// fetch APIs with given name and maturity status
def queryAPIs(apigwUrl, apiName, maturityState) {

	if (maturityState != "") {
		key = "maturityState"
		value = maturityState
	} else {
		key = "apiName"
		value = apiName
	}

	def body = """ {
	  "types": [
	    "api"
	  ],
	  "condition": "or",
	  "scope": [
	    {
	      "attributeName": "${key}",
	      "keyword": "${value}"
	    }
	  ],
	  "responseFields": [
	    "apiName",
	    "id",
	    "name",
	    "apiVersion"
	  ],
	  "from": 0,
	  "size": -1
	}"""

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/search", 
				validResponseCodes: '200'

	println("query got back: "+response.content);

	def content = readJSON file: '', text: "${response.content}"

	return content;
}

// Link given API with application i.e. Application Key
def linkApiWithApp(apigwUrl, apiRef, appRef) {

	println("linking app with api "+apiRef);

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				url: "${apigwUrl}/rest/apigateway/applications/" + appRef + "/apis", 
				validResponseCodes: '200:400'

	println("got back:" + response.content)

	def reqContent = readJSON file: '', text: "${response.content}"

	if (reqContent.apiIDs == null) {
		reqContent.apiIDs = []
	}
	
	reqContent.apiIDs = reqContent.apiIDs << apiRef

	println("will resend "+reqContent.toString())

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: reqContent.toString(), 
				url: "${apigwUrl}/rest/apigateway/applications/" + appRef + "/apis", 
				validResponseCodes: '200'

	println("query got back: "+response.content);

	def content = readJSON file: '', text: "${response.content}"

	return content;
}

def deployNewAPIToAPIGateway(apigwUrl, apiName, repoUrl, repoUser, repoPassword, version) {

	def body = """{
		"apiName": "${apiName}",
  		"apiVersion": "${version}",
  		"apiGroups": ["Demo"],
		"maturityState": "Beta",
  		"type": "swagger",
  		"url": "${repoUrl}",
  		"authorizationValue": {
    		"keyName": "Authorization",
    		"value": "${authString(repoUser,repoPassword)}",
    		"type": "header"
  		}
	}"""

	print("body: "+body)

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis", 
				validResponseCodes: '200:201'

	return response.content;
}

// maturityState "Beta"
def replaceAPIForAPIGateway(apigwUrl, id, repoUrl, repoUser, repoPassword, version, maturityState) {

	def body = """{
		"apiVersion": "${version}",
		"maturityState": "${maturityState}",
		"apiGroups": ["Demo"],
  		"type": "swagger",
  		"url": "${repoUrl}",
  		"authorizationValue": {
    		"keyName": "Authorization",
    		"value": "${authString(repoUser,repoPassword)}",
    		"type": "header"
  		}
	}"""

	print("body: "+body)

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis/${id}", 
				validResponseCodes: '200:201'

	return response.content;
}

// Duplicates existing API with new version reference
def createNewVersionForAPIForAPIGateway(apigwUrl, id, newVersion) {

	def body = """{
  		"newApiVersion": "${newVersion}",
  		"retainApplications": "true"
	}"""

	print("body: "+body)

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis/${id}/versions", 
				validResponseCodes: '200:201'

	return response.content;
}

// Deploys APIs found in git source directory to API Gateway
def deployAPIsFromGitHubToAPIGateway(apigwUrl, repoAccount, repo, repoUser, repoPassword, directory) {

	return deployAPIsToAPIGateway(apigwUrl, "https://raw.githubusercontent.com/${repoAccount}/${repo}/main/apis/", repoUser, repoPassword, directory)
}

// Deploys each API definition found in sub-directory '{directory}/src/apis' to API Gateway
// assumes that the name of the API is given in the first part of the filename postfixed with '-' e.g. "HelloWorld-api-1.0.swagger" where "HelloWorld" is the name of the API
def deployAPIsToAPIGateway(apigwUrl, swaggerEndPoint, swaggerUser, swaggerPassword, directory) {

	def dir = new File(directory)
	def refs = [];

	println("Will upload API definitions found in :"+dir.getAbsolutePath())

	def files = new File(dir, "/src/apis").list()

	files.each { file -> 
		
		content = deployAPIToAPIGateway(apigwUrl, file.split('-')[0], swaggerEndPoint + file, swaggerUser, swaggerPassword, true)

		refs = refs << (content.apiResponse.api.id)
	}

	return refs;
}

// Deploys API at given end-point to API Gateway
def deployAPIToAPIGateway(apigwUrl, apiName, swaggerEndPoint, swaggerUser, swaggerPassword, replaceCurrentVersion) {

	def version = 1
	def newVersion = 1
	def apiRef = ""

	println("Processing API with name "+apiName)

	def results = queryAPIs(apigwUrl, apiName, "")
	
	if (results != null && results.api != null) {

		def lv = -1
		results.api.each { api ->

			def v = api.apiVersion ? api.apiVersion.toInteger() : 1

			if (v > lv) {

				println("current version is " + v)

				if (v == null)
					version = 1
				else
					version = api.apiVersion.toInteger()

				apiRef = api.id
				newVersion = version + 1
				lv = version
			}
		}
	}

	def data = null

	if (newVersion == 1) {

		println("Importing as new API "+apiName+":1 (Beta) via "+swaggerEndPoint)

		data = deployNewAPIToAPIGateway(apigwUrl, apiName, swaggerEndPoint, swaggerUser, swaggerPassword, version)

		def content = readJSON file: '', text: "${data}"

		println("Will now link with app " + API_TEST_APP);

		linkApiWithApp(apigwUrl, content.apiResponse.api.id, getApplicationId(apigwUrl, API_TEST_APP));


	} else if (replaceCurrentVersion) {

		println("Importing as new version of existing API "+apiName+":"+newVersion+" (Beta) - "+apiRef + "via " + swaggerEndPoint)
		
		data = createNewVersionForAPIForAPIGateway(apigwUrl, apiRef, newVersion)

		versionResponse = readJSON file: '', text: "${data}"
		apiRef = versionResponse.apiResponse.api.id

		data = replaceAPIForAPIGateway(apigwUrl, apiRef, swaggerEndPoint, swaggerUser, swaggerPassword, newVersion, "Beta")
	
	} else {

		println("Updating existing version of API "+apiName+":"+version+" (Beta) - "+apiRef)

		data = replaceAPIForAPIGateway(apigwUrl, apiRef, swaggerEndPoint, swaggerUser, swaggerPassword, version, "Beta")
	}

	def content = readJSON file: '', text: "${data}"

	return content;
}

// repoUrl - e.g. https://github.com/johnpcarter/api-deployment.git
// check out the APIs from github, will then reference this to determine what APIs to deploy to API Gateway
def checkoutAPIs(repoAccount, repo) {

	println("Checking out from ${repoAccount} and repo ${repo}")

	def repoUrl = "https://github.com/${repoAccount}/${repo}.git"

// relativeTargetDir is related to jenkins side, doh!

	checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'src']], submoduleCfg: [], userRemoteConfigs: [[url: repoUrl]]])

	//checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'src']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: "git-apis", url: repoUrl]]])
}

def getTestStatus(apiContainer) {

	println("Checking test status of ${apiContainer}")

	def testUrl = "${apiContainer}/rad/jc.test.runner:api/ping"

	response = httpRequest acceptType: 'APPLICATION_JSON',  
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				url: testUrl, 
				validResponseCodes: '200'
				
	jsn = readJSON file: '', text: "${response.content}"
	
	return jsn.status == 'COMPLETED'
}

// Run test stub for given API, test stub needs to comply with name-space presented by ${TST_NAMESPACE}/${apiName}/${TST_POSTFIX}
def testAPI(apigwServer, testServer, apiRef) {

	apiWrapper = getAPI(apigwServer, apiRef)

	def api = apiWrapper.apiResponse.api.apiName.toLowerCase()
	def gatewayEndpoint = java.net.URLEncoder.encode(apiWrapper.apiResponse.gatewayEndPoints[0], "UTF-8")

	if (TST_POSTFIX != null) 
		testStub = "${testServer}${TST_NAMESPACE}${api}${TST_POSTFIX}"
	else 
		testStub = "${testServer}${TST_NAMESPACE}${api}"

	def url = "${testStub}/${gatewayEndpoint}"

	println("Using test stub at "+url)

	httpRequest acceptType: 'APPLICATION_JSON', 
					authentication: 'test-server', 
					contentType: 'APPLICATION_JSON', 
					httpMode: 'GET', 
					customHeaders: [[maskValue: false, name: 'api-key', value: API_TEST_APP]],
					ignoreSslErrors: true, 
					url: url, 
					validResponseCodes: '200'
					
					
}

pipeline {
	agent any
	environment {
		
		GIT_ACCOUNT='TMilliard-sag'
		GIT_REPO='devops-demo'

		APIGW_SERVER='http://devops-demo_wm-api-gateway_1:5555'
		
		API_SERVER='http://devops-demo_helloworld_1:5555'


		APIPORTAL="default"
		APIPORTAL_COMMUNITY="Public Community"
		API_TEST_APP="TestApp"
		API_STAGE="UAT"
		API_STAGE_DESCRIPTION="UAT stage description"
		API_STAGE_URL="https://pocadeo.apigw-az-eu.webmethods.io"
		API_STAGE_PROD="PROD"
		API_STAGE_PROD_DESCRIPTION="PROD stage description"
		API_STAGE_PROD_URL="http://devops-demo_wm-api-gateway-prod_1:5555"

	}
	stages {
		stage('Prepare') {
			steps {
				script {
					
					def userInput = input(
						id: 'apiInput', message: 'Git Repository', parameters: [
							[$class: 'TextParameterDefinition', defaultValue: GIT_ACCOUNT, description: 'GIT Account', name: 'apiAccount'],
							[$class: 'TextParameterDefinition', defaultValue: GIT_REPO, description: 'GIT Repository', name: 'apiRepo'],
						])

					GIT_REPO=userInput['apiRepo']
					GIT_ACCOUNT=userInput['apiAccount']
					
					def esbInput = input(
						id: 'esbInput', message: 'API & Service Containers', parameters: [
							[$class: 'TextParameterDefinition', defaultValue: API_SERVER, description: 'API Runtime Container', name: 'esbServer'],
							[$class: 'TextParameterDefinition', defaultValue: APIGW_SERVER, description: 'webMethods API Gateway', name: 'apiServer'],
							[$class: 'TextParameterDefinition', defaultValue: API_STAGE, description: 'API Gatway Stage ', name: 'apiStage'],
							[$class: 'TextParameterDefinition', defaultValue: API_STAGE_DESCRIPTION, description: 'API Gatway Stage description', name: 'apiStageDescription'],
							[$class: 'TextParameterDefinition', defaultValue: API_STAGE_PROD, description: 'API Gatway PROD Stage ', name: 'apiStageProd'],
							[$class: 'TextParameterDefinition', defaultValue: API_STAGE_PROD_DESCRIPTION, description: 'API Gatway PROD Stage Description ', name: 'apiStageProdDescription']
						])

					API_SERVER=esbInput['esbServer']
					APIGW_SERVER=esbInput['apiServer']
					API_STAGE=esbInput['apiStage'];
					API_STAGE_DESCRIPTION=esbInput['apiStageDescription'];
					API_STAGE_PROD=esbInput['apiStageProd'];
					API_STAGE_PROD_DESCRIPTION=esbInput['apiStageProdDescription'];

					def apiInput = input(
						id: 'apiInput', message: 'API Details', parameters: [
							[$class: 'TextParameterDefinition', defaultValue: API_TEST_APP, description: 'Test App Name', name: 'apiApp'],
							[$class: 'TextParameterDefinition', defaultValue: APIPORTAL, description: 'API Portal to Deploy to', name: 'apiPortal'],
							[$class: 'TextParameterDefinition', defaultValue: APIPORTAL_COMMUNITY, description: 'API Community', name: 'apiCommunity']
						])

					API_TEST_APP=apiInput['apiApp']
					APIPORTAL=apiInput['apiPortal']
					APIPORTAL_COMMUNITY=apiInput['apCommunity']

					cleanWs()

					checkoutAPIs(GIT_ACCOUNT, GIT_REPO)

					println("GIT ACCOUNT IS " + GIT_ACCOUNT);

				}
			}
		}
		stage('SetTargets') {
			// hard-coded below to keep jenkins setup simples!!
			steps {
				input("Create targets for deployments. Ready to create ?")
				script {
						try {
							STAGE_PROD_ID = createStage(APIGW_SERVER, API_STAGE_PROD, API_STAGE_PROD_DESCRIPTION, API_STAGE_PROD_URL, "Administrator", "Manage")
							createAlias(APIGW_SERVER, STAGE_PROD_ID, "API_HOST", "Back end for service", "${API_SERVER}" )
						} catch (err) {
							println("Promotion env creation error : "+err)
						}
						try {
							STAGE_ID = createStage(APIGW_SERVER, API_STAGE, API_STAGE_DESCRIPTION, API_STAGE_URL, "thierry.milliard@softwareag.com", "M@nage123")
							createAlias(APIGW_SERVER, STAGE_ID, "API_HOST", "Back end for service", "${API_SERVER}" )
						} catch (err) {
							println("Promotion env creation error : "+err)
						}						
				}
			}
		}
		stage('CreateTestApp') {
			// hard-coded below to keep jenkins setup simples!!
			/*environment {
				REPO_CREDS = credentials('git-apis') 
			}*/
			steps {
				script {
					
				}
			}
		}
		stage('ActivateGlobalPolicies') {
			steps {
				input("Deployment Completed, Ready to Test?")
				script {

/*					PROD_API_IDS = []
					FAILED_API_IDS = []

					TST_API_IDS.each { ref ->

						try {
							println("Testing API Service "+ref)
							
							getTestStatus(API_SERVER);
							
							println("Test Successful, activating API "+ref)

							activateAPI(APIGW_SERVER, ref)

							println("setting maturity level to Test")

							setAPIMaturity(APIGW_SERVER, ref, "Test");

							PROD_API_IDS = PROD_API_IDS << ref;
					
						} catch (err) {

							println("Test for API "+ref+" failed with error: "+err)
							//deactivateAPI(APIGW_SERVER, ref)

							setAPIMaturity(APIGW_SERVER, ref, "Failed");

							FAILED_API_IDS = FAILED_API_IDS << ref
						}
					}
*/					
				}
			}
		}

	}
}
