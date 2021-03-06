#!/usr/bin/env groovy

node('rhel8'){
	stage('Checkout repo') {
		deleteDir()
		git url: 'https://github.com/jboss-fuse/vscode-datavirt'
	}

	stage('Install requirements') {
		def nodeHome = tool 'nodejs-8.11.1'
		env.PATH="${env.PATH}:${nodeHome}/bin"
		sh "npm install -g typescript vsce"
	}

	stage('Build') {
		sh "npm install"
		sh "npm run vscode:prepublish"
	}

	withEnv(['JUNIT_REPORT_PATH=report.xml']) {
		stage('Test') {
			wrap([$class: 'Xvnc']) {
				sh "npm test --silent"
				junit 'report.xml'
			}
		}
	}

	stage('Package') {
		def packageJson = readJSON file: 'package.json'
		sh "vsce package -o vscode-datavirt-${packageJson.version}-${env.BUILD_NUMBER}.vsix"
		sh "npm pack && mv vscode-datavirt-${packageJson.version}.tgz vscode-datavirt-${packageJson.version}-${env.BUILD_NUMBER}.tgz"
	}

	if(params.UPLOAD_LOCATION) {
		stage('Snapshot') {
			def filesToPush = findFiles(glob: '**.vsix')
			sh "rsync -Pzrlt --rsh=ssh --protocol=28 ${filesToPush[0].path} ${UPLOAD_LOCATION}/snapshots/vscode-datavirt/"
			stash name:'vsix', includes:filesToPush[0].path
			def tgzFilesToPush = findFiles(glob: '**.tgz')
			stash name:'tgz', includes:tgzFilesToPush[0].path
			sh "rsync -Pzrlt --rsh=ssh --protocol=28 ${tgzFilesToPush[0].path} ${UPLOAD_LOCATION}/snapshots/vscode-datavirt/"
		}
	}
}

node('rhel8'){
	if(publishToMarketPlace.equals('true')){
		timeout(time:5, unit:'DAYS') {
			input message:'Approve deployment?', submitter: 'apupier,lheinema,bfitzpat,tsedmik,djelinek'
		}

		stage("Publish to Marketplace") {
			unstash 'vsix'
			unstash 'tgz'
			withCredentials([[$class: 'StringBinding', credentialsId: 'vscode_java_marketplace', variable: 'TOKEN']]) {
				def vsix = findFiles(glob: '**.vsix')
				sh 'vsce publish -p ${TOKEN} --packagePath' + " ${vsix[0].path}"
			}
			archiveArtifacts artifacts:"**.vsix,**.tgz"

			stage "Promote the build to stable"
			def vsix = findFiles(glob: '**.vsix')
			sh "rsync -Pzrlt --rsh=ssh --protocol=28 ${vsix[0].path} ${UPLOAD_LOCATION}/stable/vscode-datavirt/"
			def tgz = findFiles(glob: '**.tgz')
			sh "rsync -Pzrlt --rsh=ssh --protocol=28 ${tgz[0].path} ${UPLOAD_LOCATION}/stable/vscode-datavirt/"
		}
	}
}
