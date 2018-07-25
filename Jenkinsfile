pipeline {
    agent any
    // In additional to manual runs, trigger somewhere at midnight to
    // give us the max time in a day to get things right.
    triggers {
	// Master never runs--Feb 31st.
	cron('0 0 31 2 *')
	// Nightly @12am, for "snapshot", skip "release" night.
	//cron('0 0 2-31 * *')
	// First of the month @12am, for "release" (also "current").
	//cron('0 0 1 * *')
    }
    environment {
	// Pin dates and day to beginning of run.
	START_DATE = sh (
	    script: 'date +%Y-%m-%d',
	    returnStdout: true
	).trim()

	START_DAY = sh (
	    script: 'date +%A',
	    returnStdout: true
	).trim()
	// The branch of geneontology/go-site to use.
	TARGET_GO_SITE_BRANCH = 'iteration'
	// The people to call when things go bad. It is a comma-space
	// "separated" string.
	TARGET_ADMIN_EMAILS = 'sjcarbon@lbl.gov'
	// The file bucket(/folder) combination to use.
	TARGET_BUCKET = 'go-data-product-experimental'
	// The URL prefix to use when creating site indices.
	TARGET_INDEXER_PREFIX = 'http://experimental.geneontology.io'
	// The Zenodo concept ID to use for releases (and occasionally
	// master testing).
	ZENODO_REFERENCE_CONCEPT = '199441'
	ZENODO_ARCHIVE_CONCEPT = '212052'
	// Control make to get through our loads faster if
	// possible. Assuming we're cpu bound for some of these...
	// wok has 48 "processors" over 12 "cores", so I have no idea;
	// let's go with conservative and see if we get an
	// improvement.
	//MAKECMD = 'make --jobs --max-load 12.0'
	MAKECMD = 'make'
	// GOlr load profile.
	GOLR_SOLR_MEMORY = "128G"
	GOLR_LOADER_MEMORY = "192G"
	GOLR_INPUT_ONTOLOGIES = [
	    "http://skyhook.berkeleybop.org/master/ontology/extensions/go-gaf.owl"
	].join(" ")
	GOLR_INPUT_GAFS = [
	    //"http://www.geneontology.org/gene-associations/submission/paint/pre-submission/gene_association.paint_other.gaf",
	    "http://skyhook.berkeleybop.org/master/annotations/aspgd.gaf.gz",
	    "http://skyhook.berkeleybop.org/master/annotations/goa_chicken.gaf.gz",
	    "http://skyhook.berkeleybop.org/master/annotations/goa_chicken_complex.gaf.gz",
	    "http://skyhook.berkeleybop.org/master/annotations/goa_uniprot_all_noiea.gaf.gz",
	    "http://skyhook.berkeleybop.org/master/annotations/wb.gaf.gz"
	].join(" ")
    }
    options{
	timestamps()
    }
    stages {
	// Very first: pause for a few minutes to give a chance to
	// cancel and clean the workspace before use.
	stage('Ready and clean') {
	    steps {

		// Check that we do not affect public targets on
		// non-mainline runs.
		script {
		    if( BRANCH_NAME != 'master' && TARGET_BUCKET == 'go-data-product-experimental'){
			echo 'Only master can touch that target.'
			//sh '`exit -1`'
		    }else if( BRANCH_NAME != 'snapshot' && TARGET_BUCKET == 'go-data-product-snapshot'){
			echo 'Only master can touch that target.'
			sh '`exit -1`'
		    }else if( BRANCH_NAME != 'release' && TARGET_BUCKET == 'go-data-product-release'){
			echo 'Only master can touch that target.'
			sh '`exit -1`'
		    }
		}

		// Give us a minute to cancel if we want.
		//sleep time: 1, unit: 'MINUTES'
		cleanWs()
	    }
	}
	stage('Initialize') {
	    steps {
		// Start preparing environment.
		parallel(
		    "Report": {
			sh 'env > env.txt'
			sh 'echo $BRANCH_NAME > branch.txt'
			sh 'echo "$BRANCH_NAME"'
			sh 'cat env.txt'
			sh 'cat branch.txt'
			sh 'echo $START_DAY > dow.txt'
			sh 'echo "$START_DAY"'
		    },
		    "Reset base": {
			// Get a mount point ready
			sh 'mkdir -p $WORKSPACE/mnt || true'
			// Ninja in our file credentials from Jenkins.
			withCredentials([file(credentialsId: 'skyhook-private-key', variable: 'SKYHOOK_IDENTITY')]) {
			    // Try and ssh fuse skyhook onto our local system.
			    sh 'sshfs -o StrictHostKeyChecking=no -o IdentitiesOnly=true -o IdentityFile=$SKYHOOK_IDENTITY -o idmap=user skyhook@skyhook.berkeleybop.org:/home/skyhook $WORKSPACE/mnt/'
			}
			// Remove anything we might have left around from
			// times past.
			sh 'rm -r -f $WORKSPACE/mnt/$BRANCH_NAME || true'
			// Rebuild directory structure.
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/bin || true'
			// WARNING/BUG: needed for arachne to run at
			// this point.
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/lib || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/ttl || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/blazegraph || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/annotations || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/pages || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/solr || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/metadata || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/annotations || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/ontology || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/reports || true'
			// Tag the top to let the world know I was at least
			// here.
			sh 'echo "TODO: Note software versions." > $WORKSPACE/mnt/$BRANCH_NAME/notes.txt'
			sh 'date >> $WORKSPACE/mnt/$BRANCH_NAME/notes.txt'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/foo'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/bar'
			sh 'date > $WORKSPACE/mnt/$BRANCH_NAME/foo/date.txt'
			sh 'lsb_release > $WORKSPACE/mnt/$BRANCH_NAME/foo/thing.txt'
			// TODO: This should be wrapped in exception
			// handling. In fact, this whole thing should be.
			sh 'fusermount -u $WORKSPACE/mnt/ || true'
		    }
		)
	    }
	}
	// Build owltools and get it into the shared filesystem.
	stage('Ready production software') {
	    steps {
		parallel(
		    "Ready owltools": {
			sh 'sleep 0'
		    },
		    "Ready robot": {
			sh 'sleep 0'
		    },
		    "Ready arachne": {
			sh 'sleep 0'
		    },
		    "Ready blazegraph-runner": {
			sh 'sleep 0'
		    }
		)
	    }
	}
	// See https://github.com/geneontology/go-ontology for details
	// on the ontology release pipeline. This ticket runs
	// daily(TODO?) and creates all the files normally included in
	// a a release, and deploys to S3
	stage('Produce ontology') {
	    steps {
		sh 'sleep 0'
	    }
	}
	stage('Produce GAFs, TTLs, and journal (mega-step)') {
	    steps {
		sh 'sleep 0'
	    }
	}
	// A new step to think about. What is our core metadata?
	stage('Produce metadata') {
	    steps {
		sh 'sleep 0'
	    }
	    // WARNING: Extra safety as I expect this to sometimes fail.
	    post {
                always {
		    // Bail on the remote filesystem.
		    sh 'fusermount -u $WORKSPACE/mnt/ || true'
		    // Purge the copyover point.
		    sh 'rm -r -f $WORKSPACE/copyover || true'
                }
            }
	}
	stage('Sanity I') {
	    steps {
		sh 'sleep 0'
	    }
	    // WARNING: Extra safety as I expect this to sometimes fail.
	    post {
                always {
		    // Bail on the remote filesystem.
		    sh 'fusermount -u $WORKSPACE/mnt/ || true'
		    // Purge the copyover point.
		    sh 'rm -r -f $WORKSPACE/copyover || true'
                }
            }
	}
	//...
	stage('Produce derivatives') {
            steps {
		sh 'sleep 0'
            }
	}
	//...
	stage('Sanity II') {
	    steps {
		echo 'TODO: Sanity II'
	    }
	}
	stage('Archive') {
	    when { anyOf { branch 'release'; branch 'master'; branch 'issue-009-zenodo' } }
	    steps {
		// Experimental stanza to support mounting the sshfs
		// using the "hidden" skyhook identity.
		sh 'mkdir -p $WORKSPACE/mnt/ || true'
		withCredentials([file(credentialsId: 'skyhook-private-key', variable: 'SKYHOOK_IDENTITY')]) {
		    sh 'sshfs -oStrictHostKeyChecking=no -o IdentitiesOnly=true -o IdentityFile=$SKYHOOK_IDENTITY -o idmap=user skyhook@skyhook.berkeleybop.org:/home/skyhook $WORKSPACE/mnt/'
		}
		// Copy the product to the right location. As well,
		// archive.
		withCredentials([file(credentialsId: 'aws_go_push_json', variable: 'S3_PUSH_JSON'), file(credentialsId: 's3cmd_go_push_configuration', variable: 'S3CMD_JSON'), string(credentialsId: 'zenodo_go_sandbox_token', variable: 'ZENODO_TOKEN')]) {
		    // Ready...
		    dir('./go-site') {
			git branch: TARGET_GO_SITE_BRANCH, url: 'https://github.com/geneontology/go-site.git'

			// WARNING: Caveats and reasons as same
			// pattern above. We need this as some clients
			// are not standard and it turns out there are
			// some subtle incompatibilities with urllib3
			// and boto in some versions, so we will use a
			// virtual env to paper that over.  See:
			// https://github.com/geneontology/pipeline/issues/8#issuecomment-356762604
			sh 'python3 -m venv mypyenv'
			withEnv(["PATH+EXTRA=${WORKSPACE}/go-site/bin:${WORKSPACE}/go-site/mypyenv/bin", 'PYTHONHOME=', "VIRTUAL_ENV=${WORKSPACE}/go-site/mypyenv", 'PY_ENV=mypyenv', 'PY_BIN=mypyenv/bin']){

			    // Extra package for the indexer.
			    sh 'python3 ./mypyenv/bin/pip3 install pystache'

			    // Correct for (possibly) bad boto3,
			    // as mentioned above.
			    sh 'python3 ./mypyenv/bin/pip3 install boto3'

			    // Extra package for the uploader.
			    sh 'python3 ./mypyenv/bin/pip3 install filechunkio'

			    // Grab BDBag.
			    sh 'python3 ./mypyenv/bin/pip3 install bdbag'

			    // Need for large uploads in requests.
			    sh 'python3 ./mypyenv/bin/pip3 install requests-toolbelt'

			    // Well, we need to do a couple of things here in
			    // a structured way, so we'll go ahead and drop
			    // into the scripting mode.
			    script {

				// Build a testing version of a
				// generic BDBag/DOI workflow, keeping
				// special bucket mappings in mind.
				if( env.BRANCH_NAME == 'release' ){
				    sh 'python3 ./scripts/create-bdbag-remote-file-manifest.py -v --walk $WORKSPACE/mnt/$BRANCH_NAME/ --remote http://release.geneontology.org/$START_DATE --output manifest.json'
				}else if( env.BRANCH_NAME == 'master' ){
				    sh 'python3 ./scripts/create-bdbag-remote-file-manifest.py -v --walk $WORKSPACE/mnt/$BRANCH_NAME/ --remote $TARGET_INDEXER_PREFIX --output manifest.json'

				    // Make holey BDBag in fixed directory.
				    sh 'mkdir go-release-reference'
				    sh 'python3 ./mypyenv/bin/bdbag ./go-release-reference --remote-file-manifest manifest.json --archive tgz'

				    // Archive the holey bdbag for
				    // this run.
				    sh 'python3 ./scripts/zenodo-version-update.py --verbose --sandbox --key $ZENODO_TOKEN --concept $ZENODO_REFERENCE_CONCEPT --file go-release-reference.tgz --output ./release-reference-doi.json --revision $START_DATE'

				    // Make full BDBag (unarchived, as
				    // we want to leave it to pigz) in
				    // place in the shared filesystem
				    // directory--copying may be
				    // rather expensive at this point.
				    sh 'python3 ./mypyenv/bin/bdbag $WORKSPACE/mnt/$BRANCH_NAME'

				    // Tarball the whole directory for
				    // "deep" archive (handmade BDBag).
				    sh 'tar --use-compress-program=pigz -cvf go-release-archive.tgz -C $WORKSPACE/mnt/$BRANCH_NAME .'

				    // Archive it too.
				    sh 'python3 ./scripts/zenodo-version-update.py --verbose --sandbox --key $ZENODO_TOKEN --concept $ZENODO_ARCHIVE_CONCEPT --file go-release-archive.tgz --output ./release-archive-doi.json --revision $START_DATE'

				    // Copy the referential metadata
				    // files and DOIs to skyhook
				    // metadata/ for easy inspection,
				    // now that we have done the full
				    // BDBag in place (and we will not
				    // have to worry about having
				    // archive references in our
				    // archive).
				    sh 'cp go-release-reference.tgz $WORKSPACE/mnt/$BRANCH_NAME/metadata/go-release-reference.tgz'
				    sh 'cp manifest.json $WORKSPACE/mnt/$BRANCH_NAME/metadata/bdbag-manifest.json'
				    sh 'cp release-reference-doi.json $WORKSPACE/mnt/$BRANCH_NAME/metadata/release-reference-doi.json'
				    sh 'cp release-archive-doi.json $WORKSPACE/mnt/$BRANCH_NAME/metadata/release-archive-doi.json'
				}
			    }
			}
		    }
		}
	    }
	    // WARNING: Extra safety as I expect this to sometimes fail.
	    post {
                always {
		    // Bail on the remote filesystem.
		    sh 'fusermount -u $WORKSPACE/mnt/ || true'
		}
	    }
	}
	stage('Publish') {
	    when { anyOf { branch 'release'; branch 'snapshot'; branch 'master' } }
	    steps {
		sh 'sleep 0'
	    }
	    // WARNING: Extra safety as I expect this to sometimes fail.
	    post {
                always {
		    // Bail on the remote filesystem.
		    sh 'fusermount -u $WORKSPACE/mnt/ || true'
		}
	    }
	}
	// Big things to do on release.
	stage('Deploy') {
	    when { anyOf { branch 'release' } }
	    steps {
		sh 'sleep 0'
	    }
	    // WARNING: Extra safety as I expect this to sometimes fail.
	    post {
                always {
		    // Bail on the remote filesystem.
		    sh 'fusermount -u $WORKSPACE/mnt/ || true'
                }
            }
	}
	// stage('TODO: Final status') {
	//     steps {
	// 	echo 'TODO: final'
	//     }
	// }
	// stage('Flush') {
	//     steps {
	// 	echo 'TODO: Flush/invalidate CDN'
	//     }
	// }
    }
}
