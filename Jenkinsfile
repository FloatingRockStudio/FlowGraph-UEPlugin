String resolveSetting(script, String overrideValue, String envName, boolean required = true) {
    String value = overrideValue?.trim()
    if (!value) {
        value = script.env[envName]?.trim()
    }

    if (required && !value) {
        script.error("Missing ${envName}. Set the Jenkins environment variable or the matching override parameter.")
    }

    return value ?: ''
}

String sanitizeDescription(String value) {
    return (value ?: 'Automated Flow plugin sync')
        .replace('\r', ' ')
        .replace('\n', ' ')
        .trim()
}

pipeline {
    agent any

    options {
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        skipDefaultCheckout()
        timestamps()
    }

    parameters {
        string(name: 'UE_ENGINE_ROOT_OVERRIDE', defaultValue: '', description: 'Optional override for UE_ENGINE_ROOT.')
        string(name: 'P4_CREDENTIALS_ID_OVERRIDE', defaultValue: '', description: 'Optional override for P4_CREDENTIALS_ID used by the Jenkins P4 plugin.')
        string(name: 'P4_ENGINE_CLIENT_OVERRIDE', defaultValue: '', description: 'Optional override for P4_ENGINE_CLIENT.')
        string(name: 'P4_SUBMIT_CLIENT_OVERRIDE', defaultValue: '', description: 'Optional override for P4_SUBMIT_CLIENT. Defaults to P4_ENGINE_CLIENT when unset.')
        string(name: 'UE_PROJECT_PATH_OVERRIDE', defaultValue: '', description: 'Optional override for UE_PROJECT_PATH used for automation and game validation.')
        string(name: 'UE_VALIDATION_PLUGIN_PATH_OVERRIDE', defaultValue: '', description: 'Optional override for UE_VALIDATION_PLUGIN_PATH. Defaults to P4_PLUGIN_DEST when unset.')
        string(name: 'TEST_CATEGORY_OVERRIDE', defaultValue: '', description: 'Optional override for TEST_CATEGORY.')
        string(name: 'P4_PLUGIN_DEST_OVERRIDE', defaultValue: '', description: 'Optional override for P4_PLUGIN_DEST used for the submit workspace path.')
        string(name: 'P4_SUBMIT_DESCRIPTION_OVERRIDE', defaultValue: '', description: 'Optional Perforce submit description override.')
        booleanParam(name: 'RUN_GAME_VALIDATION', defaultValue: true, description: 'Validate that the plugin compiles for a non-editor game target after tests pass.')
        booleanParam(name: 'ENABLE_P4_SUBMIT', defaultValue: true, description: 'Allow the Perforce publish stage on the submit branch.')
    }

    environment {
        TARGET_PLATFORM = 'Win64'
        BUILD_CONFIGURATION = 'Development'
        ENGINE_EDITOR_TARGET = 'UnrealEditor'
        DEFAULT_TEST_CATEGORY = 'Flow'
        P4_SUBMIT_BRANCH = 'master'
        AUTOMATION_REPORT_ROOT = 'Saved\\Automation\\Reports'
        PLUGIN_RUNTIME_VALIDATION_ROOT = 'Saved\\PluginRuntimeValidation'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Resolve Settings') {
            steps {
                script {
                    env.PLUGIN_FILE = "${env.WORKSPACE}\\Flow.uplugin"
                    env.RESOLVED_ENGINE_ROOT = resolveSetting(this, params.UE_ENGINE_ROOT_OVERRIDE, 'UE_ENGINE_ROOT')
                    env.RESOLVED_P4_CREDENTIALS_ID = resolveSetting(this, params.P4_CREDENTIALS_ID_OVERRIDE, 'P4_CREDENTIALS_ID')
                    env.RESOLVED_P4_ENGINE_CLIENT = resolveSetting(this, params.P4_ENGINE_CLIENT_OVERRIDE, 'P4_ENGINE_CLIENT')
                    env.RESOLVED_P4_SUBMIT_CLIENT = resolveSetting(this, params.P4_SUBMIT_CLIENT_OVERRIDE, 'P4_SUBMIT_CLIENT', false) ?: env.RESOLVED_P4_ENGINE_CLIENT
                    env.RESOLVED_PROJECT_PATH = resolveSetting(this, params.UE_PROJECT_PATH_OVERRIDE, 'UE_PROJECT_PATH')
                    env.RESOLVED_P4_PLUGIN_DEST = resolveSetting(this, params.P4_PLUGIN_DEST_OVERRIDE, 'P4_PLUGIN_DEST')
                    env.RESOLVED_VALIDATION_PLUGIN_PATH = resolveSetting(this, params.UE_VALIDATION_PLUGIN_PATH_OVERRIDE, 'UE_VALIDATION_PLUGIN_PATH', false)
                    env.RESOLVED_TEST_CATEGORY = resolveSetting(this, params.TEST_CATEGORY_OVERRIDE, 'TEST_CATEGORY', false) ?: env.DEFAULT_TEST_CATEGORY

                    if (!env.RESOLVED_VALIDATION_PLUGIN_PATH?.trim()) {
                        env.RESOLVED_VALIDATION_PLUGIN_PATH = env.RESOLVED_P4_PLUGIN_DEST
                    }

                    env.BUILD_BAT = "${env.RESOLVED_ENGINE_ROOT}\\Engine\\Build\\BatchFiles\\Build.bat"
                    env.RUN_UAT_BAT = "${env.RESOLVED_ENGINE_ROOT}\\Engine\\Build\\BatchFiles\\RunUAT.bat"
                    env.UBT_EXE = "${env.RESOLVED_ENGINE_ROOT}\\Engine\\Binaries\\DotNET\\UnrealBuildTool\\UnrealBuildTool.exe"
                    env.EDITOR_CMD_EXE = "${env.RESOLVED_ENGINE_ROOT}\\Engine\\Binaries\\Win64\\UnrealEditor-Cmd.exe"
                    env.P4_DESCRIPTION = sanitizeDescription(params.P4_SUBMIT_DESCRIPTION_OVERRIDE)
                }

                bat '''
@echo off
if not exist "%PLUGIN_FILE%" (
    echo Plugin file not found at %PLUGIN_FILE%
    exit /b 1
)
if not exist "%RESOLVED_ENGINE_ROOT%" (
    echo Engine root not found at %RESOLVED_ENGINE_ROOT%
    exit /b 1
)
if not exist "%BUILD_BAT%" (
    echo Build.bat not found at %BUILD_BAT%
    exit /b 1
)
if not exist "%RUN_UAT_BAT%" (
    echo RunUAT.bat not found at %RUN_UAT_BAT%
    exit /b 1
)
if not exist "%UBT_EXE%" (
    echo UnrealBuildTool.exe not found at %UBT_EXE%
    exit /b 1
)
if not exist "%EDITOR_CMD_EXE%" (
    echo UnrealEditor-Cmd.exe not found at %EDITOR_CMD_EXE%
    exit /b 1
)
if not exist "%RESOLVED_PROJECT_PATH%" (
    echo Automation host project not found at %RESOLVED_PROJECT_PATH%
    exit /b 1
)
echo Engine root: %RESOLVED_ENGINE_ROOT%
echo Host project: %RESOLVED_PROJECT_PATH%
echo Validation plugin path: %RESOLVED_VALIDATION_PLUGIN_PATH%
'''
            }
        }

        stage('Sync Engine Workspace') {
            steps {
                p4sync(
                    charset: 'none',
                    credential: env.RESOLVED_P4_CREDENTIALS_ID,
                    populate: syncOnly(),
                    workspace: staticSpec(
                        name: env.RESOLVED_P4_ENGINE_CLIENT,
                        pinHost: false
                    )
                )
            }
        }

        stage('Compile Engine') {
            steps {
                bat '''
@echo off
call "%BUILD_BAT%" %ENGINE_EDITOR_TARGET% %TARGET_PLATFORM% %BUILD_CONFIGURATION% -WaitMutex -NoHotReloadFromIDE
'''
            }
        }

        stage('Compile Plugin') {
            steps {
                bat '''
@echo off
"%UBT_EXE%" -ModuleWithDeps -Plugin="%PLUGIN_FILE%" %TARGET_PLATFORM% %BUILD_CONFIGURATION%
'''
            }
        }

        stage('Overlay Plugin Into Validation Workspace') {
            steps {
                bat '''
@echo off
if not exist "%RESOLVED_VALIDATION_PLUGIN_PATH%" mkdir "%RESOLVED_VALIDATION_PLUGIN_PATH%"
robocopy "%WORKSPACE%" "%RESOLVED_VALIDATION_PLUGIN_PATH%" /MIR /R:2 /W:2 /NFL /NDL /NP /XD .git .vs Binaries Intermediate Saved
if %ERRORLEVEL% LEQ 7 exit /b 0
exit /b %ERRORLEVEL%
'''
            }
        }

        stage('Run Plugin Tests') {
            steps {
                bat '''
@echo off
if exist "%WORKSPACE%\\%AUTOMATION_REPORT_ROOT%" rmdir /s /q "%WORKSPACE%\\%AUTOMATION_REPORT_ROOT%"
mkdir "%WORKSPACE%\\%AUTOMATION_REPORT_ROOT%"
"%EDITOR_CMD_EXE%" "%RESOLVED_PROJECT_PATH%" -unattended -nop4 -nosplash -NullRHI -NoSound -NoSplash -TestExit="Automation Test Queue Empty" "-ExecCmds=Automation RunTests %RESOLVED_TEST_CATEGORY%; Quit" "-ReportExportPath=%WORKSPACE%\\%AUTOMATION_REPORT_ROOT%" "-ReportOutputPath=%WORKSPACE%\\%AUTOMATION_REPORT_ROOT%" "-log=%WORKSPACE%\\Saved\\Automation\\FlowAutomation.log"
'''

                junit 'Saved/Automation/Reports/**/*.xml'
            }
        }

        stage('Validate Game Runtime Build') {
            when {
                expression { return params.RUN_GAME_VALIDATION }
            }
            steps {
                bat '''
@echo off
if exist "%WORKSPACE%\\%PLUGIN_RUNTIME_VALIDATION_ROOT%" rmdir /s /q "%WORKSPACE%\\%PLUGIN_RUNTIME_VALIDATION_ROOT%"
call "%RUN_UAT_BAT%" BuildPlugin -Plugin="%PLUGIN_FILE%" -Package="%WORKSPACE%\\%PLUGIN_RUNTIME_VALIDATION_ROOT%" -TargetPlatforms=%TARGET_PLATFORM% -NoHostPlatform
'''
            }
        }

        stage('Submit To Perforce') {
            when {
                beforeAgent true
                expression {
                    return params.ENABLE_P4_SUBMIT && !env.CHANGE_ID && env.BRANCH_NAME == env.P4_SUBMIT_BRANCH
                }
            }
            steps {
                script {
                    if (env.P4_DESCRIPTION == 'Automated Flow plugin sync') {
                        env.P4_DESCRIPTION = sanitizeDescription(env.CHANGE_TITLE ?: env.CHANGE_BRANCH ?: env.GIT_BRANCH ?: '')
                    }
                }

                bat '''
@echo off
if not exist "%RESOLVED_P4_PLUGIN_DEST%" mkdir "%RESOLVED_P4_PLUGIN_DEST%"
robocopy "%WORKSPACE%" "%RESOLVED_P4_PLUGIN_DEST%" /MIR /R:2 /W:2 /NFL /NDL /NP /XD .git .vs Binaries Intermediate Saved
if %ERRORLEVEL% LEQ 7 exit /b 0
exit /b %ERRORLEVEL%
'''

                p4publish(
                    credential: env.RESOLVED_P4_CREDENTIALS_ID,
                    publish: submit(
                        delete: true,
                        description: env.P4_DESCRIPTION,
                        onlyOnSuccess: true,
                        reopen: false
                    ),
                    workspace: staticSpec(
                        name: env.RESOLVED_P4_SUBMIT_CLIENT,
                        pinHost: false
                    )
                )
            }
        }
    }

    post {
        always {
            archiveArtifacts allowEmptyArchive: true, artifacts: 'Saved/Automation/**/*'
        }
    }
}