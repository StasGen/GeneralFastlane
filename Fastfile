#!/usr/bin/ruby

fastlane_version "2.99.1"
default_platform :ios

#####################################################
# Public lanes
#####################################################

desc "Download dSYM files from iTC and upload them to Crashlytics"
desc "you can specify `version`, latest will be used if not provided"
lane :refresh_dsyms do |options|
  download_dsyms(version: options[:version] || "latest")
  upload_symbols_to_crashlytics
  clean_build_artifacts
end


#####################################################
# Private lanes
#####################################################

desc "Run unit tests"
private_lane :execute_test do
  reset_git_repo
  clear_derived_data
  update_pods
  scan
  slather
end


desc "Submit a new Crashlytics (Stage server) build from provided/develop branch with badge"
private_lane :execute_stage do |options|
  branch = options[:branch]
  checkout_if_needed(options[:builder], branch)

  build_crashlytics(branch || git_branch, "stage", options[:build_number])
end


desc "Submit a new Crashlytics (Prod server) from provided/develop branch with badge"
private_lane :execute_prod do |options|
  branch = options[:branch]
  checkout_if_needed(options[:builder], branch)

  build_crashlytics(branch || git_branch, "prod", options[:build_number])
end


desc "Submit a new AdHoc Build to Apple TestFlight from RC branch"
private_lane :execute_release_appstore do |options|
  slack_train_start(distance: 5, train: "🚀", reverse_direction: true, rail: "✨")
  
  ensure_git_status_clean
  slack_train
  
  git_checkout "rc"
  slack_train
  
  update_pods
  slack_train
  
  fullfill_plists_with_configs
  slack_train
  
  build_appstore_release
  slack_train
  
  testflight
  slack_train
  
  refresh_dsyms(version:get_version_number_from_plist(scheme: ENV["APPSTROE_SCHEME"]))
  slack_train
  
  clean_build_artifacts
  reset_git_repo
  slack_train
  
  post_to_slack(message: "Build is processed and ready to be tested #{get_version}")
end


desc "Checks if all configs settings exists in *.info plists"
private_lane :execute_fullfill_plists_with_configs do
  config = Xcodeproj::Config.new(ENV["XCODE_COFIG_PATH"])
  project = Xcodeproj::Project.open(ENV["XCODE_PROJ_PATH"])

  project.targets.each do |target|
    plist_path = "../#{target.build_configurations[0].build_settings["INFOPLIST_FILE"]}"
    plist = Xcodeproj::Plist.read_from_path(plist_path)
    config.attributes.each do |key, value|
      plist[key] = "${#{key}}"
    end
    Xcodeproj::Plist.write_to_path(plist, plist_path)
  end
end


desc "Passes firebase analytics argument on launchtime"
private_lane :execute_enable_firebase_debug_mode do
  project_path = ENV["XCODE_PROJ_PATH"]
  schemes = Xcodeproj::Project.schemes(project_path)
  
  schemes.each do |value|
    path = project_path + "/xcshareddata/xcschemes/" + value + ".xcscheme"
    sheme = Xcodeproj::XCScheme.new(path)
    sheme.launch_action.command_line_arguments = Xcodeproj::XCScheme::CommandLineArguments.new([{ :argument => '-FIRAnalyticsDebugEnabled', :enabled => true }])
    sheme.save!
  end
end


desc "Create a new release on GitHub from master branch, add tag from project version and upload changelog for it"
private_lane :execute_create_tag do
  ensure_git_status_clean
  git_checkout("master")
  git_pull
  
  version_number = get_version_number_from_plist(scheme: ENV["APPSTROE_SCHEME"])
  puts "Last git tag is #{last_git_tag}"
  raise "This version is already tagged!" if git_tag_exists(tag: "v.#{version_number}")
  puts "New git tag is v.#{version_number}"

  github_release = set_github_release(
    repository_name: ENV["GITHUB_REPOSITORY_NAME"],
    api_token: ENV["GITHUB_TOKEN"],
    name: "Version #{version_number}",
    tag_name: "v.#{version_number}",
    description: (File.read("changelog.txt") rescue "No changelog provided"),
    commitish: "master"
  )
end


#####################################################
# Helpers
#####################################################

def get_version
  return get_version_number(target: ENV["MAIN_TARGET"])
end


def checkout_if_needed(builder, branch)
  if builder != "jenkins" && branch != nil
    reset_git_repo
    git_checkout branch
  end
end


def upload_time
  return Time.now.strftime("%d/%m/%Y %H:%M")
end


def update_pods
  cocoapods(
    clean: true,
    try_repo_update_on_error: true
  )
end


last_slack_progress = "[-----:train:]"

def post_to_slack(options)
  progress = options[:progress]
  last_slack_progress = progress
  slack(
    message: "#{progress} #{options[:message]}",
    slack_url: ENV["SLACK_CHANNEL_URL"],
    success: options[:success],
    default_payloads: []
  )
end

def git_checkout(branch)
  command = [
    'git',
    'checkout',
    branch
  ]
  Actions.sh(command.join(' '))
  UI.important "Successfully checkout branch #{remote_branch}."
end

#####################################################
# Crashlytics Beta deploy
#####################################################

def build_crashlytics(branch, env, build_number)
  UI.header "Building for branch: #{git_branch} environment: #{env}"
  ensure_git_status_clean
  set_build_number_if_needed(build_number)
  clear_derived_data
  update_pods
  fullfill_plists_with_configs
  enable_firebase_debug_mode
  set_adhoc_provisioning_profiles
  commit_hash = last_git_commit[:abbreviated_commit_hash]
  message = nil
  
  case env
  when "stage"
    add_badge(shield: "stage-#{get_version}-orange", no_badge: true)
    gym(
      scheme: "Stage",
      export_method: "ad-hoc",
      configuration: "ReleaseStage"
    )
    message = "🎉 Stage build is ready to be tested #{get_version}, #{branch} 🎉"
    notes = "Branch: #{branch}\nHash: #{commit_hash}\nUpload time: #{upload_time}\nServer: Staging"
  when "prod"
    add_badge(shield: "prod-#{get_version}-orange", no_badge: true)
    gym(
      scheme: "Prod",
      export_method: "ad-hoc",
      configuration: "ReleaseProd"
    )
    message = "🍷 Prod build is ready to be tested #{get_version}, #{branch} 🥃"
    notes = "Branch: #{branch}\nHash: #{commit_hash}\nUpload time: #{upload_time}\nServer: Prod"
  else
    "You gave me #{env} -- I have no idea what to do with that."
  end
  
  crashlytics_upload(notes)
  clean_build_artifacts
  reset_git_repo
  post_to_slack(message: message)
end


def set_build_number_if_needed(build_number)
  if build_number != nil
    UI.header "Set build number: #{build_number}"
    increment_build_number_in_plist(
      build_number: build_number,
      target: ENV["MAIN_TARGET"]
    )
  end
end


def set_adhoc_provisioning_profiles
  update_app_identifier(
    plist_path: ENV["PLIST_PATH"],
    xcodeproj: ENV["XCODE_PROJ"]
  )

  update_provisioning_profile_specifier(
    target: ENV["MAIN_TARGET"],
    new_specifier: ENV["PROVISION_PROFILE_DISTRIBUTION"]
  )
end


def crashlytics_upload(notes)
  crashlytics(
    api_token: ENV["CRASHLYTICS_API_TOKEN"],
    build_secret: ENV["CRASHLYTICS_BUILD_SECRET"],
    emails: ENV["CRASHLYTICS_EMAILS"],
    groups: nil,
    notes: notes, 
    notifications: true
  )
  upload_symbols_to_crashlytics
end


#####################################################
# Appstore deploy
#####################################################

def build_appstore_release
  set_appstore_provisioning_profiles

  version_number = get_version_number_from_plist(scheme: ENV["APPSTROE_SCHEME"])
  version_current = latest_testflight_build_number(version: version_number).to_i + 1
  version_live_raw = latest_live_version_build_number_if_exists

  version_live = version_live_raw.to_i + 1
  version = version_live > version_current ? version_live : version_current
  
  increment_build_number_in_plist(
    build_number: version.to_s,
    target: ENV["MAIN_TARGET"]
  )
  gym(
    scheme: ENV["APPSTROE_SCHEME"],
    export_method: "app-store",
    configuration: "ReleaseAppStore"
  )
end

def set_appstore_provisioning_profiles
  update_app_identifier(
    plist_path: ENV["PLIST_PATH"],
    xcodeproj: ENV["XCODE_PROJ"]
  )
  update_provisioning_profile_specifier(
    target: ENV["MAIN_TARGET"],
    new_specifier: ENV["PROVISION_PROFILE_APPSTORE"]
  )
end

def latest_live_version_build_number_if_exists
	Spaceship::Tunes.login(CredentialsManager::AppfileConfig.try_fetch_value(:apple_id))
	app = Spaceship::Tunes::Application.find(CredentialsManager::AppfileConfig.try_fetch_value(:app_identifier))
  	latest_version = app.live_version || app.latest_version
  	version = latest_version.version
  	build_number = latest_version.build_version ||= "1"
  	return build_number
end

#####################################################
# Error exception
#####################################################

error do |lane, exception|
  clean_build_artifacts
  reset_git_repo
  UI.important "Fail? with '#{lane}' Exception #{exception} "
  if lane == :execute_release_appstore
    slack_train_crash
  end
  if lane == :execute_stage
    post_to_slack(message: "Failed to deliver stage build #{exception}", success: false)
  end
  if lane == :execute_prod
    post_to_slack(message: "Failed to deliver stage build #{exception}", success: false)
  end
end
