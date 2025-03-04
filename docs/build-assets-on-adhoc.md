# Building assets on adhoc environment
## Flow of CI build system
- `bin/start-build`
  - signal a cron job on daemon servers (e.g. adhoc, staging, test) to restart the build on that server by creating `./build-started` file

- `cookbooks/cdo-apps/templates/default/crontab.erb`
  - Run `aws/ci_build` every minute on daemon environments.

- `aws/ci_build`
  - Fetch and pull git changes. Run bundle install. Run `bundle exec rake ci`
  - `lib/cdo/rake_utils.rb` contains git, npm, rake helper functions

- `lib/rake/ci.rake`
  - Run Firebase CI (:upload_rules, :set_config, :clear_test_channels)
  - Build apps, dashboard, pegasus and tools
  - Deploy: Upgrade frontend and update CloudFormation stack
  - Flush cache
  - Publish GitHub release if on production

- `lib/rake/build.rake`
  - Build apps (first), dashboard, pegasus, and tools (last)

  - Build apps
    - Only build if any of the apps_build_trigger_paths have changed since last build.
    - Install dependencies by running `yarn`
    - Rebuild PhantomJS `npm rebuild phantomjs-prebuilt`
    - Then `npm run build` or `npm run build:dist` (minified/uglified version) depends on the environment
      - Execute grunt build tasks as set in `apps/package.json`, e.g. `grunt clean build`
      - `apps/Gruntfile.js` confgiures and delegates work to the various webpack tasks
      - `apps/webpack.js` bundles javascript files and minifies them. Results are in `apps/build/package/js/`
    - Write file `build/commit_hash` when done

  - Build dashboard
    - Run bundle install in dashboard dir
    - Migrate database `bundle exec rake db:setup_or_migrate`
      - `dashboard/lib/tasks/setup_or_migrate.rake`
    - Update schema cache file (only on non-prod environments) `bundle exec rake db:schema:cache:dump` and `bundle exec rake db:round_trip_schema_cache`
    - Seed dashboard `bundle exec rake seed:all`
      - `dashboard/lib/tasks/seed.rake`
      - Can skip this on development env by setting `skip_seed_all: true` in locals.yml
    - Clean dashboard asset `bunlde exec rake assets:clean`
    - Precompile dashboard assets `bundle exec rake assets:precompile`
      - This rake task does a few things
        - Generates/compiles all the static assets (including images, css, js, etc.) using the rails asset pipeline
        - Copies over all the files that were generated by webpack, adding a unique digest hash to the filenames, and constructing a manifest file with the mapping
        - Results are in `dashboard/public/assets/`
      - `dashboard/lib/tasks/asset_sync.rake` runs 2 other actions after runing assets:precompile
        - `assets:precompile_application_js` precompile application.js with js_compressor (:uglifier)
        - `assets:sync` synchronizes newly-added assets to S3 if CDN deployment is enabled
      - Configurations for the asset pipeline (config.assets lines)
        - `dashboard/config/application.rb`
        - `dashboard/config/environments/production.rb`
    - Upgrade dashboard `sudo service dashboard upgrade`

- More details on building apps and asset pipeline: [apps/docs/build.md](https://github.com/code-dot-org/code-dot-org/blob/staging/apps/docs/build.md)

## How to debug asset pipeline issue on adhoc
- Check build log for issue. (The system auto rebuild whenever we switch branch.)
  - `tail -f log/chat_messages.log`
  - Example of a successful build
    ```
    [adhoc] Running CI build...
    [adhoc] Uploading security rules to firebase...
    [adhoc] Setting firebase configuration parameters...
    [adhoc] Updating local <b>chef</b> cookbooks...
    [adhoc] Running rake build...
    [adhoc] Running build...
    [adhoc] Installing <b>apps</b> dependencies...
    [adhoc] Building <b>apps</b>...
    [adhoc] Installing <b>dashboard</b> bundle...
    [adhoc] Migrating <b>dashboard</b> database...
    [adhoc] Seeding <b>dashboard</b>...
    [adhoc] Cleaning <b>dashboard</b> assets...
    [adhoc] Precompiling <b>dashboard</b> assets...
    [adhoc] Upgrading <b>dashboard</b>.
    [adhoc] Installing <b>pegasus</b> bundle...
    [adhoc] Updating <b>pegasus</b> database...
    [adhoc] Upgrading <b>pegasus</b>.
    [adhoc] build succeeded in 11:15 minutes
    [adhoc] rake build succeeded in 11:15 minutes
    [adhoc] Running CloudFormation stack update...
    [adhoc] [stack update] update stack: adhoc-ha-adhoc...
    [adhoc] [stack update] No updates are to be performed.
    [adhoc] CloudFormation stack update succeeded in 0:07 minutes
    [adhoc] Running Flush cache...
    [adhoc] Flush cache succeeded in 0:09 minutes
    [adhoc] CI build succeeded in 11:35 minutes
    ```
- Fix the issue and rebuild by running `bin/start-build`. If everything works, asset changes should show up correctly.
- Alternative option:
  - Manually re-try the failed step and follow next steps in `lib/rake/build.rake` to re-build apps and dashboard.
  - Upgrade dashboard service. May need to restart varnish
    - sudo service dashboard upgrade
    - sudo service varnish restart

## How to fix seeding issue when building adhoc
- Option 1: Fix seeding file  (e.g. ApSchoolCode seeding)
  - Go to `dashboard/lib/tasks/seed.rake` to see how that table is seeded. It could be seeded from a CSV or S3.
  - Edit the seed CSV file to remove conflicting rows. Make a backup of the file first.
- Option 2: Skip seeding
  - Edit `lib/rake/build.rake` -> task dashboard -> comment out `seed:all` call
