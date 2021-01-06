import jenkins.model.Jenkins
import hudson.model.*
import java.text.SimpleDateFormat

pipeline {
	environment {
        // Use credentials() to hide the environment variable's output
		// In case jppf server and jppf node on one machine, use localhost
		server = "10.134.22.153"
		jppf_server = "10.134.22.153:11111"
		jppf_server_jmx = "10.134.22.153:11198"					
		currentdate= "${formatDate("MMddyyyy")}"							
		local_data_center = "C:\\Share\\CICD"
		remote_data_center = "\\\\10.134.22.153\\Share\\CICD"
		autolog_location = "\\\\10.134.22.153\\Share\\LogFiles_Jenkins\\${BUILD_NUMBER}"
		softwares_location = "${remote_data_center}\\Softwares"
		jppf_location = "${remote_data_center}\\JPPF\\Official"					
		ta_queue_location = "${local_data_center}\\TADispatcher"
		default_batch_location = "${local_data_center}\\TA_Batch"
        //repotoken = credentials('webframeworkci-git-ssh')
		mail_list = "${Recipients}"
    }

	agent { label 'AutomationHost' }

    stages {
		stage("Prepare Automation batch files and requred folders") {			
			steps {
				script {	
					//env.currentdate = todayTime.format("MMddyyyy")
					echo currentdate
					env.folderstamp = "${currentdate}_${BUILD_NUMBER}"
					echo folderstamp
					//echo "Cloning automation-testing repo to local"
					//bat '@echo off && cd C:/Temp && del /F /Q automation-testing && rmdir /F /Q automation-testing && git clone https://git.openearth.community/WebFramework/automation-testing.git'
					//echo "Backup the TA repo"
					//bat 'cd "C:/Program Files/LogiGear/TestArchitect/binclient" && java -jar TAImportExportTool.jar --ExportRepository --server "DS365" --port "53400" --repoName BoreholeDM --destinationFile C:/Temp/automation-testing [--overwrite<true|false>]'
					echo "Step: Recognize the app to set global environment and modify TA batch file"
					if (MainApp_URL.contains("search")) {
						appType = "Search_App"
					} else {
						appType = "Admin_App"
					}
					
					if(Test_Suite != "Custom"){
						Batch_File_Location = "TestSuites\\${appType}\\${Test_Suite}"
					}else {
						Batch_File_Location = "TestSuites\\Custom\\${Batch_File_Location}"
					}
					declareTAsettings(appType)
					ta_batch_location = default_batch_location + "\\" + Batch_File_Location
					local_batch_location = "${WORKSPACE}\\TA_Batch\\${appType}\\${folderstamp}\\main"
					precondition_batch = "${default_batch_location}\\TestSuites\\${appType}\\Precondition"
					echo "${local_batch_location}"
					local_precondition_batch = "${WORKSPACE}\\TA_Batch\\${appType}\\${folderstamp}\\Precondition"
					echo "${local_precondition_batch}"
					// Handle batch file
					echo "Create local batch file folder if not exist..."
					bat '@echo off && if not exist "' + local_batch_location + '" md "' + local_batch_location + '"'
					bat '@echo off && if not exist "' + local_precondition_batch + '" md "' + local_precondition_batch + '"'
					echo "Copying batch files from data center..."
					bat '@echo off && copy "' + ta_batch_location + '\\*.bat" ' + local_batch_location
					bat '@echo off && copy "' + precondition_batch + '\\*.bat" ' + local_precondition_batch

					echo "Step: Add Execution info to to TA batch files"
					echo "Creating local results folder"
					bat '@echo off && if not exist "' + export_result_location + '" md "' + export_result_location +'"'
					echo "Adding startup setting"
					bat '@echo off && md "' + modified_batch_location + '" "' + modified_precondition + '" "' + master_report_folder + '"'
					addSettingsToBatchFiles("**\\TA_Batch\\${appType}\\${folderstamp}\\Precondition\\*.*",result_setting + ' ' + startup_setting + ' ' + repo_result_upload_setting,"${modified_precondition}")
					addSettingsToBatchFiles("**\\TA_Batch\\${appType}\\${folderstamp}\\main\\*.*",result_setting + ' ' + startup_setting + ' ' + repo_result_upload_setting,"${modified_batch_location}") 
					// Handle if it is a downstream run
					// if (UpStream_Build_No != "NA") {
					// 	upstream_flag = "\"${local_data_center}\\running_upstream.flag\""
					// 	//fileEx = fileExists "running_upstream.flag"
					// 	def fileEx = bat (
					// 		returnStdout: true,
					// 		script: "@echo off && if not exist ${upstream_flag} echo false")
					// 	echo "Flag file exist:" + fileEx							
					// 	// If this is second run, it will continue. Otherwise, terminate the run
					// 	if (fileEx.trim() == "false") {												
					// 		bat 'echo UpStream>' + upstream_flag
					// 		error("This is not the main downstream job. Aborted!")	
					// 	} else {
					// 		bat "del /F /Q ${upstream_flag}"
					// 		echo "Run from upstream job no ${UpStream_Build_No}"
					// 	}
					// }
					//echo "Starting JPPF driver"
					
				}
			}
		}

		// Install Softwares
		stage("Install Prerequisite softwares And Start JPPF on Test machines") {
			agent {label 'TestsRunner'}	
			steps {
				script {					
					def precondition_jobs = [:]
					echo "${WORKSPACE}"
					slaves = nodeNames('TestsRunner')					
					for (int l = 0; l < slaves.size() ; l++) {  
						slave = slaves[l]
						precondition_jobs["slave_${slave}"] = {
							echo "Step: Extract Sources"				
							bat '@echo off && powershell -command "Expand-Archive -Force .\\Softwares.zip ." && powershell -command "Expand-Archive -Force .\\JPPF-node.zip ." && powershell -command "Expand-Archive -Force .\\BrowserInstallers.zip ." && powershell -command "Expand-Archive -Force .\\harness_9.802.100.zip ."'
							if (Install_Prerequisite_Softwares == "true") {					
								echo "Step: Install Softwares"						
								echo "Installing TA"
								bat "@echo off && AutoInstall_TA\\install.bat|exit 0"
								echo "Installing Python"
								bat "@echo off && AutoInstall_Python\\install.bat|exit 0"
								echo "Installing Chrome"
								bat "@echo off && wmic product where \"name like '%%Google Chrome%%'\" call uninstall /nointeractive||exit 0"
								bat "@echo off && BrowserInstallers\\chrome_installer.exe /silent /install|exit 0"
							}

							echo "Step: Start jppf"
							int total_jppfthreads = No_Of_JPPF_Threads.toInteger()
							echo "Total threads: " + total_jppfthreads.toString()
							env.ClientWorkspace=(bat (
								returnStdout: true,
								script: "@echo off && echo %CD%")).trim()
							echo "Workspace is ${ClientWorkspace}"
							echo "Workspace default is ${WORKSPACE}"
							echo "Killing running TA services and jppf services"
							bat "@echo off && wmic process where \"CommandLine like '%%jppf%%'\" call terminate"
							bat "@echo off && wmic process where 'name=\"TACTRL.exe\"' call terminate"
							bat "@echo off && wmic process where 'name=\"TAHotKeys.exe\"' call terminate"
							echo "Starting jppf services"													
							for (int i = 0; i<total_jppfthreads; i++) {
								bat "@echo off && schtasks.exe /create /sc MINUTE /mo 5 /tn startJPPF /tr \"${ClientWorkspace}\\startJppf.bat ${ClientWorkspace}\\JPPF-node\" /f"
								bat "@echo off && schtasks.exe /run /tn startJPPF"
								bat "@echo off && schtasks.exe /delete /tn startJPPF /f"
							}

							echo "Step: Import TA backup to local repo"
							bat "@echo off && cd \"C:/Program Files/LogiGear/TestArchitect/binclient\" && java -jar TAImportExportTool.jar --ImportRepository --server localhost --port 53404 --repoName DS365 --filePath ${ClientWorkspace}\\DS365.dat --overwrite true"
      					}
					}												
					parallel precondition_jobs																				
				}					
			}			
		}
					
		/* This stage prepare environment by doing: 
			- All running jobs are terminated
			- All node machines do not have any running automation tests
			- All automation batch files are added HTML result export variable
		*/

		stage("Deploy and Monitor Testing"){
			steps {
				timeout(time: 720, unit: 'MINUTES'){
					script {
						echo "Step: Start Job"
						echo "Submitting cancel job to jppf..."
						// Run precondition
						if (Run_Precondition == "true") {
							bat '@echo off && cd /d ' + ta_queue_location + '&& run.bat cancel --name "' + preconditon_job + '" --server "' + jppf_server_jmx + '" || exit 0'
							echo "Submit job to server"
							bat '@echo off && cd /d ' + ta_queue_location + '&& run.bat create --name "' + preconditon_job + '" --server "' + jppf_server + '" --batch "' + modified_precondition + '" --split "yes" --broadcast "no"'
							echo "Wait for job running completed"
							bat '@echo off && cd /d ' + ta_queue_location + '&& run.bat wait --server "' + jppf_server_jmx + '" --name "' + preconditon_job + '" -t "3480"' + '|| exit 0'
						}
						// Run main batch
						bat '@echo off && cd /d ' + ta_queue_location + '&& run.bat cancel --name "' + jppf_job + '" --server "' + jppf_server_jmx + '" || exit 0'
						echo "Submit job to server"
						bat '@echo off && cd /d ' + ta_queue_location + '&& run.bat create --name "' + jppf_job + '" --server "' + jppf_server + '" --batch "' + modified_batch_location + '" --split "yes" --broadcast "no"'
						echo "Step: Wait for job running completedly"
						bat '@echo off && cd /d ' + ta_queue_location + '&& run.bat wait --server "' + jppf_server_jmx + '" --name "' + jppf_job + '" -t "3480"'
					}
					
				}
			}
		}   

		stage("Reporting"){
			steps{
				script{
					try{
						echo "Step: Generate excel report"
						bat '@echo off && md "' + result_toreport_folder + '" "' + excel_report_folder + '" "' + local_email_folder + '"'
						bat '@echo off && copy ' + export_result_location + "\\*.html " + result_toreport_folder
						bat '@echo off && cd ' + excelReportTool + '&& collect_reports.bat ' + result_toreport_folder + ' ' + excel_report_folder
						bat '@echo off && rename ' + excel_report_folder + "\\Web*.xlsx Regression_${folderstamp}.xlsx"
						bat '@echo off && copy ' + excel_report_folder + "\\Regression_${folderstamp}.xlsx " + master_report_folder

						echo "Step: Generate Email report"
						// , returnStdout:true
						def number_of_html_files = bat (
							returnStdout: true,
							script: "@echo off && cd ${export_result_location}&&dir /A-D /B|FIND /C /V \"\"")
						echo 'Number of html results: ' + number_of_html_files						
						echo "Generating HTML report"
						bat "@echo off && powershell -Command \"(gc ${summaryRepTool}\\XMLConfig_original.xml) -replace '<localHost><', '<localHost>${export_result_location_xml}\\TransformedHTML\\<' | Out-File -encoding ASCII ${summaryRepTool}\\XMLConfig.xml\""
						bat "@echo off && powershell -Command \"(gc ${summaryRepTool}\\XMLConfig.xml) -replace '<remoteHost><', '<remoteHost>http://webframework.ci.openearth.io/job/AutomationTesting/job/DispatchTests/${BUILD_NUMBER}/Automation_20Report/TAResults<' | Out-File -encoding ASCII ${summaryRepTool}\\XMLConfig.xml\""
						bat "@echo off && java -jar ${summaryRepTool}\\GenerateSummaryReport.jar ${export_result_location_xml} ${local_email_folder} || exit 0"
						bat "@echo off && md ${local_email_folder}\\TAResults && copy ${export_result_location}\\*.html ${local_email_folder}\\TAResults"
						bat "@echo off && rename ${local_email_folder}\\Email*.html Email.html"
						bat "@echo off && copy ${local_email_folder}\\*.html ${master_report_folder}"

						echo "Step: Send email and upload Automation Report to Jenkins"
						def htmlFiles = getFiles("email\\${folderstamp}\\*.html")
						if(htmlFiles.size() > 0){
							def fileName = htmlFiles[0].name
							def content = getFileContent("${local_email_folder}\\${fileName}") 
							echo "Sending Email report"
							emailext(
								//attachmentsPattern: "**/Regression_${folderstamp}.xlsx",
								subject: "${email_tittle}",
								to: mail_list,
								mimeType: "text/html",
								body: content
							)
							echo "Publishing Reports to Jenkins"
							publishHTML (target : [allowMissing: false,
								alwaysLinkToLastBuild: true,
								keepAll: true,
								reportDir: "email\\${folderstamp}",
								//reportDir: "email\\report",
								reportFiles: "**\\*.html",
								reportName: 'Automation Report',
								reportTitles: 'Automation Report'])
						}
						else{
							echo "Error: htmlFile size is < 0. Mail is not send"
						}
					}
					catch(Exception ex){
						echo "Reporting Stage Failed!!!"
						echo "Caught: ${ex}"
					}
				}
			}
		}
	}  

    post{
    	cleanup {
			node('AutomationHost') { 
				script {
					echo "Clean up Step"				
				}
			}
			node('TestsRunner') {
				script {	
					echo "Clean up is skipped"				
					// Kill all current node threads
					bat "@echo off && wmic process where \"CommandLine like '%%jppf%%'\" call terminate"
					bat "@echo off && wmic process where 'name=\"TACTRL.exe\"' call terminate"	
					bat "@echo off && wmic process where 'name=\"TAHotKeys.exe\"' call terminate"				
				}
			}
    	}
    	aborted{
			node('AutomationHost') { 
				script {
					abort_user = getAbortUser()
					echo "Build Aborted!!!"
					echo 'Email Build Aborted information'
					emailext (attachLog: true, body: "Build Aborted", compressLog: true, subject: "ABORTED by ${abort_user}: Job ${env.JOB_NAME} [" + build_number + "] ",  to: mail_list)
					echo "Submitting cancel job to jppf..."
					//bat 'cd /d ' + ta_queue_location + '&& run.bat cancel --name "SearchApp_Automation_Testing" --server "' + jppf_server_jmx + '" || exit 0'					
				}
			}
		}
    	failure{
			node('AutomationHost') {	
				script {
					echo "Build Failed!!!"		
					echo "Email Build Failed information"  			
					emailext (attachLog: true, subject: "FAILURE: Job ${env.JOB_NAME} [" + build_number + "]: ",  to: mail_list)	
				}
			}
		}
    }
}

def formatDate(String strFormat) {
	def todayTime = new Date()
	return todayTime.format(strFormat)
}
@NonCPS
def getFileContent(String filepath) {
	def fileContent = readFile(filepath)
	return fileContent
}

@NonCPS
def declareTAsettings(def appType) {
	echo "The application is " + appType
	export_result_location = "C:\\Share\\CICD\\TA_Results\\${appType}\\${folderstamp}"
	export_result_location_xml = "C:\\Share\\CICD\\TA_Results\\${appType}\\${folderstamp}\\xml"
	html_export_location = "//${server}/Share/CICD/TA_Results/${appType}/${folderstamp}"	
	xml_export_location = "//${server}/Share/CICD/TA_Results/${appType}/${folderstamp}/xml"				
	modified_batch_location = "${local_data_center}\\modified_TA_Batch\\${appType}\\${folderstamp}\\main"
	modified_precondition = "${local_data_center}\\modified_TA_Batch\\${appType}\\${folderstamp}\\precondition"
	// Variable to add to TA batch file
	repo_result_upload_setting = '-up "/BoreholeDM/Results/Regresion/CICD/' + Batch_File_Location.replace('\\','/') + '/' + folderstamp + '"'
	result_setting = '-html "' + html_export_location + '" -subfld "0" -subhtml "1" -xml "' + xml_export_location +'" -cc "Failed;Warning/Error" -cl "0" -htmlscrn "2"'
	// Report and Email tools settings
	summaryRepTool = '"' + local_data_center + '\\GenerateSummaryReport"'
	excelReportTool = local_data_center + '\\CollectResultReports'
	result_toreport_folder = excelReportTool + '\\Results\\' +  Batch_File_Location.replace('\\','\\\\') + "\\${folderstamp}"
	excel_report_folder = excelReportTool + '\\Report\\' +  Batch_File_Location.replace('\\','\\\\') + "\\${folderstamp}"
	master_report_folder = "${local_data_center}\\FinalReport\\" + Batch_File_Location.replace('\\','\\\\') + "\\${folderstamp}"
	if (appType=="Search_App") {
		startup_setting = '-ss "main_app_link=uds=' + MainApp_URL + ';keyloak_app_link=uds=' + Keycloak_App + ';app_type=uds='+ appType +';api_link=uds='+ API_URL +';jenkins_build_no=uds=' + BUILD_NUMBER + '"'
	}else {
		startup_setting = '-ss "main_app_link=uds=' + MainApp_URL + ';keyloak_app_link=uds=' + Keycloak_App + ';app_type=uds='+ appType +';api_link=uds=undefined;jenkins_build_no=uds=' + BUILD_NUMBER + '"'
	}
	local_email_folder = "${WORKSPACE}\\email\\${folderstamp}"
	local_send_email = "email\\${folderstamp}"
	email_tittle = "Test Results - ${appType} - Build ${BUILD_NUMBER}"
	//JPPF settings
	jppf_job = "${appType}_Automation_Testing"
	preconditon_job = "PreCondition_${appType}"	
}

@NonCPS
def getFiles(def wildcard){
	pwd()
	return findFiles(glob: wildcard)
}

@NonCPS
def stopPreviousRun(strBuild_url, intNumber){
	for (int t = 1; t < intNumber; t++) {
		int _t= BUILD_NUMBER.toInteger() - t
		message = _t.toString()
		bat 'curl -i -X POST "' + strBuild_url + message + '/stop" -u lam.nguyen:123Fr@mew0rk'
	} 
}

def addSettingsToBatchFiles(strWorkspaceBatchPath, strSetting, strWritePath){
	// fslWindows = findFiles(glob: "**\\TA_Batch\\SearchApp\\${folderstamp}\\main\\*.*")
	// fslCondition = findFiles(glob: "**\\TA_Batch\\SearchApp\\${folderstamp}\\precondition\\*.*")
	fslWindows = findFiles(glob: "${strWorkspaceBatchPath}")
	echo "Total batch files:" + fslWindows.size().toString()
	for (int t = 0; t < fslWindows.size(); t++) {
		def _t = t
		string content = readFile(fslWindows[_t].toString())
		if(content.contains("-html") == false){
			echo "Start replace file number " + (t + 1).toString()
			content = content.replaceAll('run.bat"', 'run.bat" ' + strSetting)
			writeFile file: "${strWritePath}\\" + fslWindows[_t].name.toString(), text: content
			echo "Writing file ${strWritePath}\\" + fslWindows[_t].name.toString()
		}
	} 
}
//Test
@NonCPS
def getAbortUser()
{
    def causee = ''
    def actions = currentBuild.getRawBuild().getActions(jenkins.model.InterruptedBuildAction)
    for (action in actions) {
        def causes = action.getCauses()

        // on cancellation, report who cancelled the build
        for (cause in causes) {
            causee = cause.getUser().getDisplayName()
            cause = null
        }
        causes = null
        action = null
    }
    actions = null

    return causee
}

@NonCPS
def nodeNames(label) {
    def nodes = []
    jenkins.model.Jenkins.instance.computers.each { c ->
        if (c.node.labelString.contains(label)) {
			if (c.isOnline().toString() == "true") {
            	nodes.add(c.node.selfLabel.name)
			}
        }
    }   
    return nodes
}

@NonCPS
def startjppf(label) {
    def nodes = []
    jenkins.model.Jenkins.instance.computers.each { c ->
        if (c.node.labelString.contains(label)) {
			if (c.isOnline().toString() == "true") {
            	nodes.add(c.node.selfLabel.name)
			}
        }
    }   
    return nodes
}

