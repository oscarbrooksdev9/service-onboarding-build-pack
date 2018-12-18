#!groovy.
import groovy.json.JsonOutput
import java.text.SimpleDateFormat
import groovy.transform.Field

/**Process** of service-onboarding-build-pack
-identify service type
-identify runtime
-checkout template
-make change to service-name in deployment-env.yml
-create repo
-add remote to git
-git push template to git
*/

// To be replaced as @Field def repo_credential_id = "value" for repo_credential_id, repo_base and repo_core
@Field def repo_credential_id = "jazz_repocreds"
@Field def repo_base = "18.235.7.234"
@Field def repo_core = "slf"
@Field def scm_type = "gitlab"

@Field def configModule
@Field def configLoader
@Field def scmModule
@Field def events
@Field def serviceMetadataLoader
@Field def service_config
@Field def utilModule

@Field def g_svc_admin_cred_ID = 'SVC_ADMIN'
@Field def auth_token = ''
@Field def g_base_url = ''
@Field def context_map = [:]

node  {
	jazzBuildModuleURL = getBuildModuleUrl()
	loadBuildModules(jazzBuildModuleURL)

	def jazz_prod_api_id = utilModule.getAPIIdForCore(configLoader.AWS.API["PROD"])
	g_base_url = "https://${jazz_prod_api_id}.execute-api.${configLoader.AWS.REGION}.amazonaws.com/prod"

	auth_token = getAuthToken()

	def service_id = params.service_id
	def admin_group = "admin_group"
	def repo_name
	def service_template

	echo "Onboarding Started with Params: $params"

	stage('Input Validation')
	{

		if (service_id) {
			service_config = serviceMetadataLoader.loadServiceMetadata(service_id)
		} else {
			error "Service Id is not available."
		}

		scmModule.setServiceConfig(service_config)
		if (!events) { error "Can't load events module"	} //Fail here
		events.initialize(configLoader, service_config, "SERVICE_CREATION", "", "", g_base_url + "/jazz/events")

		repo_name = "${service_config['domain']}_${service_config['service']}"
		context_map = [created_by : service_config['created_by']]

		events.sendStartedEvent("VALIDATE_INPUT", 'input validation starts', context_map)
		if (service_config['type'] == "api" || service_config['type'] == "lambda" || service_config['type'] == "function" || service_config['type'] == "website") {
			if (service_config['runtime'] == "nodejs" || service_config['runtime'] == "python" || service_config['runtime'] == "java" || service_config['runtime'] == "n/a") {

				switch (service_config['type']) {
					case "api":
						if (service_config['runtime'] == "nodejs") {
							service_template = "api-template-nodejs"
						}
						else if (service_config['runtime'] == "python") {
							service_template = 'api-template-python'
						}
						else if (service_config['runtime'] == "java") {
							service_template = 'api-template-java'
						}
						break

					case "lambda":
					case "function":
						if (service_config['runtime'] == "nodejs") {
							service_template = "lambda-template-nodejs"
						}
						else if (service_config['runtime'] == "python") {
							service_template = 'lambda-template-python'
						}
						else if (service_config['runtime'] == "java") {
							service_template = 'lambda-template-java'
						}
						break

					case "website":
						service_template = 'static-website-template'
						break

				}

			} else {
				send_status_email("FAILED")
				events.sendFailureEvent('VALIDATE_INPUT', "Invalid runtime: ${service_config['runtime']}", context_map)
				error "Invalid runtime: ${service_config['runtime']}"
			}

		} else {
			send_status_email("FAILED")
			events.sendFailureEvent('VALIDATE_INPUT', "Invalid service type", context_map)
			error "Invalid service type"
		}
		events.sendCompletedEvent('VALIDATE_INPUT', 'input validation completed', context_map)
	}


	stage('Get Service Template'){
		try {
			try {
				sh 'rm -rf *'
				sh 'rm -rf .*'
			}
			catch (error) {
				//do nothing
			}

			echo "service_template : $service_template"
			events.sendStartedEvent("CLONE_TEMPLATE", 'cloning template starts', context_map)
			sh 'mkdir ' + service_template
			dir(service_template)
			{
				checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: repo_credential_id, url: scmModule.getTemplateUrl(service_template)]]])
				def config = LoadConfiguration()
				def serviceMetadataJson = [
					"securityGroupIds": configLoader.AWS.SECURITY_GROUP_IDS,
					"subnetIds": configLoader.AWS.SUBNET_IDS,
					"iamRoleARN": configLoader.AWS.ROLEID
				]
				for (item in config) {
					serviceMetadataJson[item.key] = item.value
				}
				for (item in service_config.catalog_metadata) {
					serviceMetadataJson[item.key] = item.value
				}

				def repo_url = scmModule.getRepoUrl(repo_name)
				context_map['repository'] = repo_url
				context_map['metadata'] = serviceMetadataJson
			}
			events.sendCompletedEvent('CLONE_TEMPLATE', 'cloning template completed', context_map)
		}
		catch (ex) {
			send_status_email("FAILED")
			events.sendFailureEvent('CLONE_TEMPLATE', ex.getMessage(), context_map)
			error ex.getMessage()
		}
	}

	stage('Update Service Template')
	{
		try {
			dir(service_template)
			{
				events.sendStartedEvent("MODIFY_TEMPLATE", 'modify template starts', context_map)

        //Clearing depoyment-env.yml
				sh "echo -n > ./deployment-env.yml"
				// Add service_id  in deployment-env.yml
				sh "echo 'service_id: $service_id' >> ./deployment-env.yml"
			}
			events.sendCompletedEvent('MODIFY_TEMPLATE', 'modify template completed', context_map)
		}
		catch (ex) {
			send_status_email("FAILED")
			events.sendFailureEvent('MODIFY_TEMPLATE', ex.getMessage(), context_map)
			error ex.getMessage()
		}
	}

	stage('Uploading templates to code repository')
	{
		dir(service_template)
		{
			try {
				events.sendStartedEvent("CREATE_SERVICE_REPO", 'service repo creation starts', context_map)
				scmModule.createProject(service_config['created_by'], repo_name)
				events.sendCompletedEvent('CREATE_SERVICE_REPO', 'modify template completed', context_map)
			}
			catch (ex) {
				send_status_email("FAILED")
				events.sendFailureEvent('CREATE_SERVICE_REPO', ex.getMessage(), context_map)
				error ex.getMessage()
			}
		}

		withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: repo_credential_id, passwordVariable: 'PWD1', usernameVariable: 'UNAME']]) {
			def encoded_password = URLEncoder.encode(PWD1, "utf-8")

			def repo_clone_base_url = scmModule.getRepoCloneBaseUrl(repo_name)
			def repo_protocol = scmModule.getRepoProtocol()

			sh "git config --global user.email \"" + configLoader.JAZZ.ADMIN + "\""
			sh "git config --global user.name $UNAME"
			sh "git clone ${repo_protocol}$UNAME:$encoded_password@${repo_clone_base_url}"
		}
		try {
			sh "mv -nf " + service_template + "/* " + repo_name + "/"
			sh "mv -nf " + service_template + "/.* " + repo_name + "/"
		}
		catch (ex) {
			//do nothing
		}


		dir(repo_name)
		{
			sh "ls -lart"

        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: repo_credential_id, passwordVariable: 'PWD', usernameVariable: 'UNAME']]) {
				try {
					events.sendStartedEvent("ADD_WEBHOOK", 'service repo creation starts', context_map)
					scmModule.addWebhook(repo_name, "notify-events", "${g_base_url}/jazz/scm-webhook")
					events.sendCompletedEvent('ADD_WEBHOOK', 'modify template completed', context_map)
				}
				catch (ex) {
					send_status_email("FAILED")
					events.sendFailureEvent('ADD_WEBHOOK', ex.getMessage(), context_map)
					error ex.getMessage()
				}

				try {
					events.sendStartedEvent("PUSH_TEMPLATE_TO_SERVICE_REPO", 'push template to repo started', context_map)
					sh "git add --all"
					sh "git commit -m 'Code from the standard template'"
					sh "git remote -v"
					sh "git push -u origin master "
					events.sendCompletedEvent('PUSH_TEMPLATE_TO_SERVICE_REPO', 'push template completed', context_map)
				}
				catch (ex) {
					send_status_email("FAILED")
					events.sendFailureEvent('PUSH_TEMPLATE_TO_SERVICE_REPO', ex.getMessage(), context_map)
					error ex.getMessage()
				}

				try {
					events.sendStartedEvent("ADD_WRITE_PERMISSIONS_TO_SERVICE_REPO", 'adding write permission to repo started', context_map)
					scmModule.setRepoPermissions(service_config['created_by'], repo_name, admin_group)
					events.sendCompletedEvent('ADD_WRITE_PERMISSIONS_TO_SERVICE_REPO', 'adding write permission to repo started', context_map)
				}
				catch (ex) {
					send_status_email("FAILED")
					events.sendFailureEvent('ADD_WRITE_PERMISSIONS_TO_SERVICE_REPO', ex.getMessage(), context_map)
					error ex.getMessage()
				}

				try {
					events.sendStartedEvent("LOCK_MASTER_BRANCH", 'set branch permission to repo started', context_map)
					scmModule.setBranchPermissions(repo_name)
				}
				catch (ex) {
					send_status_email("FAILED")
					events.sendFailureEvent('LOCK_MASTER_BRANCH', ex.getMessage(), context_map)
					error ex.getMessage()
				}
				send_status_email("COMPLETED")
				events.sendCompletedEvent('LOCK_MASTER_BRANCH', 'creation completed', context_map)
			}
		}
	}
}
/**
 * For getting token to access catalog APIs.
 * Must be a service account which has access to all services
 */
def getAuthToken() {

	withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: g_svc_admin_cred_ID, passwordVariable: 'PWD', usernameVariable: 'UNAME']]) {

		def loginUrl = g_base_url + "/jazz/login"
		def login_json = []

		login_json = [
			'username': UNAME,
			'password': PWD
		]
		def tokenJson_token = null
		def payload = JsonOutput.toJson(login_json)

		try {
			def token = sh(script: "curl --silent -X POST -k -v \
				-H \"Content-Type: application/json\" \
					$loginUrl \
				-d \'${payload}\'", returnStdout:true).trim()

			def tokenJson = jsonParse(token)
			tokenJson_token = tokenJson.data.token

			return tokenJson_token
		}
		catch (e) {
			error "error occured: " + e.getMessage()
		}
	}

}

def LoadConfiguration() {
	def result = readFile('deployment-env.yml').trim()
	echo "result of yaml parsing....$result"
	def prop = [:]
	def resultList = result.tokenize("\n")

    // delete commented lines
    def cleanedList = []
    for (i in resultList) {
        if (i.toLowerCase().startsWith("#")) { } else {
            cleanedList.add(i)
        }
    }
    // echo "result of yaml parsing after clean up....$cleanedList"
    for (item in cleanedList) {
        // Clean up to avoid issues with more ":" in the values
		item = item.replaceAll(" ", "").replaceFirst(":", "#");
		def eachItemList = item.tokenize("#")
        //handle empty values
        def value = null;
        if (eachItemList[1]) {
            value = eachItemList[1].trim();
        }

        if (eachItemList[0]) {
            prop.put(eachItemList[0].trim(), value)
        }

    }
	echo "Loaded configurations....$prop"
	return prop
}

/**
* Send email to the recipient with the build status and any additional text content
* Supported build status values = STARTED, FAILED & COMPLETED
* @return
*/
def send_status_email (build_status) {
   	echo "Sending build notification to ${service_config['created_by']}"
	def body_subject = ''
	def body_text = ''
	def cc_email = ''
	def body_html = ''

	if (build_status == 'STARTED') {
		echo "email status started"
		body_subject = "Jazz Build Notification: Creation STARTED for service:  ${service_config['service']} "
   	} else if (build_status == 'FAILED') {
		echo "email status failed"
		def build_url = env.BUILD_URL + 'console'
		body_subject = "Jazz Build Notification: Creation FAILED for service:  ${service_config['service']} "
		body_text = body_text + '\n\nFor more details, please click this link: ' + build_url
   	} else if (build_status == 'COMPLETED') {
		body_subject = "Jazz Build Notification: Creation COMPLETED successfully for service:  ${service_config['service']} "
   	} else {
		echo "Unsupported build status, nothing to email.."
		return
   	}

	if (service_config['domain'] != '') {
		body_text = "${body_text} \n\n For Service: ${service_config['service']}  in Domain:  ${service_config['domain']}"
	}

	def fromStr = 'Jazz Admin <' + configLoader.JAZZ.ADMIN + '>'
	body = JsonOutput.toJson([
		from : fromStr,
		to : service_config['created_by'],
		subject : body_subject,
		text : body_text,
		cc : cc_email,
		html : body_html
	])

   	try {
		def sendMail = sh(script: "curl -X POST \
						${g_base_url}/jazz/email \
						-k -v -H \"Authorization: $g_login_token\" \
						-H \"Content-Type: application/json\" \
						-d \'${body}\'", returnStdout: true).trim()
		def responseJSON = parseJson(sendMail)

		if (responseJSON.data) {
			echo "successfully sent e-mail to ${service_config['created_by']}"
		} else {
			echo "exception occured while sending e-mail: $responseJSON"
		}
   	} catch (e) {
       	echo "Failed while sending build status notification"
   	}
}

@NonCPS
def parseJson(jsonString) {
    def lazyMap = new groovy.json.JsonSlurper().parseText(jsonString)
    def m = [:]
    m.putAll(lazyMap)
    return m
}

/*
* Load build module
*/

def loadBuildModules(buildModuleUrl){
	echo "loading env variables, checking repos..."

	dir('build_modules') {
		checkout([$class: 'GitSCM', branches: [
			[name: '*/master']
		], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [
				[credentialsId: repo_credential_id, url: buildModuleUrl]
			]])

		def resultJsonString = readFile("jazz-installer-vars.json")
		configModule = load "config-loader.groovy"
		configLoader = configModule.initialize(resultJsonString)
		echo "config loader loaded successfully."

		scmModule = load "scm-module.groovy"
		scmModule.initialize(configLoader)
		echo "SCM module loaded successfully."

		events = load "events-module.groovy"
		echo "Event module loaded successfully."

		serviceMetadataLoader = load "service-metadata-loader.groovy"
		serviceMetadataLoader.initialize(configLoader)
		echo "Service metadata loader module loaded successfully."

		utilModule = load "utility-loader.groovy"
	}
}

def getBuildModuleUrl() {
    if (scm_type && scm_type != "bitbucket") {
		// right now only bitbucket has this additional tag scm in its git clone path
		return "http://${repo_base}/${repo_core}/jazz-build-module.git"
    } else {
		return "http://${repo_base}/scm/${repo_core}/jazz-build-module.git"
    }
}
@NonCPS
def jsonParse(jsonString) {
    def nonLazyMap = new groovy.json.JsonSlurperClassic().parseText(jsonString)
    return nonLazyMap
}
