#!groovy

// ВНИМАНИЕ:
// Jenkins и его ноды нужно запускать с кодировкой UTF-8
//      строка конфигурации для запуска Jenkins
//      <arguments>-Xrs -Xmx256m -Dhudson.lifecycle=hudson.lifecycle.WindowsServiceLifecycle -Dmail.smtp.starttls.enable=true -Dfile.encoding=UTF-8 -jar "%BASE%\jenkins.war" --httpPort=8080 --webroot="%BASE%\war" </arguments>
//
//      строка для запуска нод
//      @"C:\Program Files (x86)\Jenkins\jre\bin\java.exe" -Dfile.encoding=UTF-8 -jar slave.jar -jnlpUrl http://localhost:8080/computer/slave/slave-agent.jnlp -secret XXX
//      подставляйте свой путь к java, порту Jenkins и секретному ключу
//
// Если запускать Jenkins не в режиме UTF-8, тогда нужно поменять метод cmd в конце кода, применив комментарий к методу

// node("artbear") {
node("slave") {
  isUnix = isUnix();
  stage('clean workspace') {
        // Wipe the workspace so we are building completely clean
        deleteDir()
  }
  stage('Получение исходных кодов') {

    checkout scm
    cmd('git config --local core.longpaths true', isUnix);

    //env.LOGOS_CONFIG="logger.rootLogger=DEBUG"; // включение отладки продукта //env.RUNNER_ENV="debug";

    cmd('git submodule update --init', isUnix)

    echo "Текущий каталог"
    echo pwd()

    echo "Проверка выполнения oscript -version - находится ли он в PATH?"
    timestamps {
        cmd("where oscript", isUnix)
        cmd("oscript -version", isUnix)
    }


    // echo "Установка свежих версий зависимостей библиотек oscript"
    timestamps {
        //cmd("opm update -all", isUnix)
         cmd("opm install", isUnix)
    }
  }

  stage('BDD тестирование'){ 

    def command = """opm run test""";
    if(env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'develop') {
        command = """opm run coverage"""
    } 
    
    def errors = []
    timestamps {
        try{
            cmd(command, isUnix)
        } catch (e) {
            errors << "BDD status : ${e}"
        }
    }

    if (errors.size() > 0) {
        currentBuild.result = 'UNSTABLE'
        for (int i = 0; i < errors.size(); i++) {
            echo errors[i]
        }
    }           

    step([$class: 'ArtifactArchiver', artifacts: '**/bdd-exec.xml', fingerprint: true])
    step([$class: 'JUnitResultArchiver', testResults: '**/bdd*.xml'])
  }

    stage('build'){
        timestamps {
            command = """opm build ./"""
            cmd(command, isUnix)
            step([$class: 'ArtifactArchiver', artifacts: '**/vanessa-runner*.ospx', fingerprint: true])
        }
    }

  stage('Контроль технического долга'){ 

    if (env.QASONAR) {
        timestamps {
        println env.QASONAR;
        def sonarcommand = "@\"./../../tools/hudson.plugins.sonar.SonarRunnerInstallation/Main_Classic/bin/sonar-scanner\""
        withCredentials([[$class: 'StringBinding', credentialsId: env.SonarOAuthCredentianalID, variable: 'SonarOAuth']]) {
            sonarcommand = sonarcommand + " -Dsonar.host.url=https://sonar.silverbulleters.org -Dsonar.login=${env.SonarOAuth}"
        }
        
        // Get version - в модуле 'src/Модули/ПараметрыСистемы.os' должна быть строка формата Версия = "0.8.1";
        def configurationText = readFile encoding: 'UTF-8', file: 'src/Модули/ПараметрыСистемы.os'
        def configurationVersion = (configurationText =~ /Версия\s*=\s*\"([^"]*)\"/)[0][1]
        sonarcommand = sonarcommand + " -Dsonar.projectVersion=${configurationVersion}"

        def makeAnalyzis = true
        if (env.BRANCH_NAME == "develop") {
            echo 'Analysing develop branch'
        } else if (env.BRANCH_NAME.startsWith("release/")) {
            sonarcommand = sonarcommand + " -Dsonar.branch=${BRANCH_NAME}"
        } else if (env.BRANCH_NAME.startsWith("PR-")) {
            // Report PR issues           
            def PRNumber = env.BRANCH_NAME.tokenize("PR-")[0]
            def gitURLcommand = 'git config --local remote.origin.url'
            def gitURL = ""
            
            if (isUnix()) {
                gitURL = sh(returnStdout: true, script: gitURLcommand).trim() 
            } else {
                gitURL = bat(returnStdout: true, script: gitURLcommand).trim() 
            }
            
            def repository = gitURL.tokenize("/")[2] + "/" + gitURL.tokenize("/")[3]
            repository = repository.tokenize(".")[0]
            withCredentials([[$class: 'StringBinding', credentialsId: env.GithubOAuthCredentianalID, variable: 'githubOAuth']]) {
                sonarcommand = sonarcommand + " -Dsonar.analysis.mode=issues -Dsonar.github.pullRequest=${PRNumber} -Dsonar.github.repository=${repository} -Dsonar.github.oauth=${env.githubOAuth}"
            }
        } else {
            makeAnalyzis = false
        }

        if (makeAnalyzis) {
            if (isUnix()) {
                sh '${sonarcommand}'
            } else {
                bat "${sonarcommand}"
            }
        }
        }
    } else {
        echo "QA runner not installed"
    }
  }
  
}

def cmd(command, isUnix) {
    // при запуске Jenkins не в режиме UTF-8 нужно написать chcp 1251 вместо chcp 65001
    if (isUnix) { sh "${command}" } else { bat "chcp 65001\n${command}"}
}
