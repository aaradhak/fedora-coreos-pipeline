node {
    checkout scm
    // these are script global vars
    pipeutils = load("utils.groovy")
    pipecfg = pipeutils.load_pipecfg()
}

properties([
    pipelineTriggers([]),
    parameters([
      choice(name: 'STREAM',
             choices: pipeutils.get_streams_choices(pipecfg),
             description: 'CoreOS stream to test'),
      string(name: 'VERSION',
             description: 'CoreOS Build ID to test',
             defaultValue: '',
             trim: true),
      string(name: 'ARCH',
             description: 'Target architecture',
             defaultValue: 'x86_64',
             trim: true),
      string(name: 'KOLA_TESTS',
             description: 'Override tests to run',
             defaultValue: "",
             trim: true),
      string(name: 'COREOS_ASSEMBLER_IMAGE',
             description: 'Override the coreos-assembler image to use',
             defaultValue: "quay.io/coreos-assembler/coreos-assembler:main",
             trim: true),
      string(name: 'SRC_CONFIG_COMMIT',
             description: 'The exact config repo git commit to run tests against',
             defaultValue: '',
             trim: true),
    ]),
    buildDiscarder(logRotator(
        numToKeepStr: '100',
        artifactNumToKeepStr: '30'
    )),
    durabilityHint('PERFORMANCE_OPTIMIZED')
])

currentBuild.description = "[${params.STREAM}][${params.ARCH}] - ${params.VERSION}"

def stream_info = pipecfg.streams[params.STREAM]

// Use eastus region for now
def region = "eastus"

def s3_stream_dir = pipeutils.get_s3_streams_dir(pipecfg, params.STREAM)

// Go with a higher memory request here because we download/decompress/upload the image
def cosa_memory_request_mb = 1792


cosaPod(memory: "${cosa_memory_request_mb}Mi", kvm: false,
        image: params.COREOS_ASSEMBLER_IMAGE,
        serviceAccount: "jenkins") {
    timeout(time: 75, unit: 'MINUTES') {
    try {

        def azure_image_name, azure_image_filepath
        lock(resource: "kola-cloud-buildfetch") {
            stage('Fetch Metadata/Image') {
                def commitopt = ''
                if (params.SRC_CONFIG_COMMIT != '') {
                    commitopt = "--commit=${params.SRC_CONFIG_COMMIT}"
                }
                // Grab the metadata. Also grab the image so we can upload it.
                withCredentials([file(variable: 'AWS_CONFIG_FILE',
                                    credentialsId: 'aws-build-upload-config')]) {
                    def (url, ref) = pipeutils.get_source_config_for_stream(pipecfg, params.STREAM)
                    def variant = stream_info.variant ? "--variant ${stream_info.variant}" : ""
                    shwrap("""
                    cosa init --branch ${ref} ${commitopt} ${variant} ${url}
                    time -v cosa buildfetch --build=${params.VERSION} --arch=${params.ARCH} \
                        --url=s3://${s3_stream_dir}/builds --artifact=azure
                    """)
                    pipeutils.withXzMemLimit(cosa_memory_request_mb - 512) {
                        shwrap("cosa decompress --build=${params.VERSION} --artifact=azure")
                    }
                    azure_image_filepath = shwrapCapture("""
                    cosa meta --build=${params.VERSION} --arch=${params.ARCH} --image-path azure
                    """)
                }

                // Use a consistent image name for this stream in case it gets left behind
                azure_image_name = "kola-fedora-coreos-${params.STREAM}-${params.ARCH}.vhd"
            }
        }

        withCredentials([file(variable: 'AZURE_KOLA_TESTS_CONFIG',
                              credentialsId: 'azure-kola-tests-config')]) {

            def azure_testing_resource_group = pipecfg.clouds?.azure?.test_resource_group
            def azure_testing_storage_account = pipecfg.clouds?.azure?.test_storage_account
            def azure_testing_storage_container = pipecfg.clouds?.azure?.test_storage_container
            def azure_testing_gallery = pipecfg.clouds?.azure?.test_gallery

            stage('Upload/Create Image') {
                // Create the image in Azure
                shwrap("""
                # First delete the blob/image since we re-use it.
                ore azure delete-gallery-image --log-level=INFO         \
                    --azure-credentials \${AZURE_KOLA_TESTS_CONFIG}     \
                    --azure-location $region                            \
                    --resource-group $azure_testing_resource_group      \
                    --gallery-name $azure_testing_gallery               \
                    --gallery-image-name $azure_image_name
                ore azure delete-blob --log-level=INFO                  \
                    --azure-credentials \${AZURE_KOLA_TESTS_CONFIG}     \
                    --azure-location $region                            \
                    --resource-group $azure_testing_resource_group      \
                    --storage-account $azure_testing_storage_account    \
                    --container $azure_testing_storage_container        \
                    --blob-name $azure_image_name
                # Then create them fresh
                ore azure upload-blob --log-level=INFO                  \
                    --azure-credentials \${AZURE_KOLA_TESTS_CONFIG}     \
                    --azure-location $region                            \
                    --resource-group $azure_testing_resource_group      \
                    --storage-account $azure_testing_storage_account    \
                    --container $azure_testing_storage_container        \
                    --blob-name $azure_image_name                       \
                    --file ${azure_image_filepath}
                # Create a fresh gallery image. Note that this will
                # create the testing gallery if it does not exist, but
                # it will be a no-op on gallery creation given the
                # gallery exists in the specified region.
                ore azure create-gallery-image --log-level=INFO         \
                    --arch $params.ARCH                                 \
                    --azure-credentials \${AZURE_KOLA_TESTS_CONFIG}     \
                    --azure-location $region                            \
                    --resource-group $azure_testing_resource_group      \
                    --gallery-name $azure_testing_gallery               \
                    --gallery-image-name $azure_image_name              \
                    --image-blob "https://${azure_testing_storage_account}.blob.core.windows.net/${azure_testing_storage_container}/${azure_image_name}"
                """)
            }

            // Since we don't have permanent images uploaded to Azure we'll
            // skip the upgrade test.
            try {
                def azure_subscription = shwrapCapture("jq -r .subscription \${AZURE_KOLA_TESTS_CONFIG}")
                kola(cosaDir: env.WORKSPACE, parallel: 10,
                     build: params.VERSION, arch: params.ARCH,
                     extraArgs: params.KOLA_TESTS,
                     skipUpgrade: true,
                     skipKolaTags: stream_info.skip_kola_tags,
                     platformArgs: """-p=azure                               \
                         --azure-credentials \${AZURE_KOLA_TESTS_CONFIG}     \
                         --azure-location $region                            \
                         --azure-disk-uri /subscriptions/${azure_subscription}/resourceGroups/${azure_testing_resource_group}/providers/Microsoft.Compute/galleries/${azure_testing_gallery}/images/${azure_image_name}/versions/1.0.0""")
            } finally {
                parallel "Delete Image": {
                    // Delete the image in Azure
                    shwrap("""
                    ore azure delete-gallery-image --log-level=INFO         \
                        --azure-credentials \${AZURE_KOLA_TESTS_CONFIG}     \
                        --azure-location $region                            \
                        --resource-group $azure_testing_resource_group      \
                        --gallery-name $azure_testing_gallery               \
                        --gallery-image-name $azure_image_name
                    ore azure delete-blob --log-level=INFO                  \
                        --azure-credentials \${AZURE_KOLA_TESTS_CONFIG}     \
                        --azure-location $region                            \
                        --resource-group $azure_testing_resource_group      \
                        --storage-account $azure_testing_storage_account    \
                        --container $azure_testing_storage_container        \
                        --blob-name $azure_image_name
                    """)
                }, "Garbage Collection": {
                    shwrap("""
                    ore azure gc --log-level=INFO                           \
                        --azure-credentials \${AZURE_KOLA_TESTS_CONFIG}     \
                        --azure-location $region
                    """)
                }
            }

        }

        currentBuild.result = 'SUCCESS'

    } catch (e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (currentBuild.result != 'SUCCESS') {
            pipeutils.trySlackSend(message: ":azure: kola-azure #${env.BUILD_NUMBER} <${env.BUILD_URL}|:jenkins:> <${env.RUN_DISPLAY_URL}|:ocean:> [${params.STREAM}][${params.ARCH}] (${params.VERSION})")
        }
    }
}} // cosaPod and timeout finish here
