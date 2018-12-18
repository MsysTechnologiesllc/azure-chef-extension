
# To create a zip file for publishing handler?

# To run tests (rspec)?

# May be task to publish the release-version of zip to azure using azure apis (release automation)

require "rake/packagetask"
require "uri"
require "net/http"
require "json"
require "zip"
require "date"
require "nokogiri"
require "mixlib/shellout"
require "./lib/chef/azure/helpers/erb.rb"

PACKAGE_NAME = "ChefExtensionHandler".freeze
MANIFEST_NAME = "publishDefinitionXml".freeze
EXTENSION_VERSION = "1.0".freeze
CHEF_BUILD_DIR = "pkg".freeze
PESTER_VER_TAG = "2.0.4".freeze # we lock down to specific tag version
PESTER_GIT_URL = "https://github.com/pester/Pester.git".freeze
PESTER_SANDBOX = "./PESTER_SANDBOX".freeze
CHEF_URL = "http://www.chef.io/about".freeze

GOV_REGIONS = ["USGov Iowa", "USGov Arizona", "USGov Texas", "USGov Virginia"].freeze

PREVIEW = "deploy_to_preview".freeze
PRODUCTION = "deploy_to_production".freeze
GOV = "deploy_to_gov".freeze
DELETE_FROM_PREVIEW = "delete_from_preview".freeze
DELETE_FROM_PRODUCTION = "delete_from_production".freeze
DELETE_FROM_GOV = "delete_from_gov".freeze
CONFIRM_INTERNAL = "confirm_internal_deployment".freeze
CONFIRM_PUBLIC = "confirm_public_deployment".freeze
DEPLOY_INTERNAL = "deploy_to_internal".freeze
DEPLOY_PUBLIC = "deploy_to_public".freeze

INTERNAL_OR_PUBLIC = [CONFIRM_INTERNAL, CONFIRM_PUBLIC].freeze
DEPLOY_TYPE = [PREVIEW, PRODUCTION, GOV].freeze
DELETE_TYPE = [DELETE_FROM_PREVIEW, DELETE_FROM_PRODUCTION, DELETE_FROM_GOV].freeze

# Hashes {src : dest} for files to be packaged
LINUX_PACKAGE_LIST = {
  "ChefExtensionHandler/*.sh" => "#{CHEF_BUILD_DIR}/",
  "ChefExtensionHandler/bin/*.sh" => "#{CHEF_BUILD_DIR}/bin",
  "ChefExtensionHandler/bin/*.py" => "#{CHEF_BUILD_DIR}/bin",
  "ChefExtensionHandler/bin/*.rb" => "#{CHEF_BUILD_DIR}/bin",
  "ChefExtensionHandler/bin/chef-client" => "#{CHEF_BUILD_DIR}/bin",
  "*.gem" => "#{CHEF_BUILD_DIR}/gems",
  "ChefExtensionHandler/HandlerManifest.json.nix" => "#{CHEF_BUILD_DIR}/HandlerManifest.json}",
}.freeze

WINDOWS_PACKAGE_LIST = {
  "ChefExtensionHandler/*.cmd" => "#{CHEF_BUILD_DIR}/",
  "ChefExtensionHandler/bin/*.bat" => "#{CHEF_BUILD_DIR}/bin",
  "ChefExtensionHandler/bin/*.ps1" => "#{CHEF_BUILD_DIR}/bin",
  "ChefExtensionHandler/bin/*.psm1" => "#{CHEF_BUILD_DIR}/bin",
  "ChefExtensionHandler/bin/*.rb" => "#{CHEF_BUILD_DIR}/bin",
  "ChefExtensionHandler/bin/chef-client" => "#{CHEF_BUILD_DIR}/bin",
  "*.gem" => "#{CHEF_BUILD_DIR}/gems",
  "ChefExtensionHandler/HandlerManifest.json" => "#{CHEF_BUILD_DIR}/HandlerManifest.json",
}.freeze

ENVIRONMENT_VARIABLES = {
  "azure_extension_cli" => "Path of azure-extension-cli binary. Download it from https://github.com/Azure/azure-extensions-cli/releases",
  "SUBSCRIPTION_ID" => "Subscription ID of the GOV Account from where extension is to be published.",
  "SUBSCRIPTION_CERT" => "Path to the Management Certificate",
  "MANAGEMENT_URL" => "Management URL for Public/Gov Cloud (e.g. https://management.core.windows.net/)",
  "EXTENSION_NAMESPACE" => "Publisher namespace (Chef.Bootstrap.WindowsAzure)",
}.freeze

PUBLISH_ENV_VARS = { "publishsettings" => "Publish settings file for Azure." }.freeze

def error_and_exit!(message)
  puts "\nERROR: #{message}\n"
  exit
end

def confirm!(type)
  puts "Do you wish to proceed?(y/n)"
  proceed = STDIN.gets.chomp() == "y"
  if not proceed
    puts "Exiting #{type} request."
    exit
  end
end

###############################################################################
# Following options were extracted from azure-extensions-cli version v1.2.7.
# We will use them for our rake tasks.
# azure-extensions-cli: Designed for Microsoft internal extension publishers to
# release, update and manage Virtual Machine extensions.
###############################################################################

# Creates an XML file used to publish or update extension.
def new_extension_manifest_opts(env_package, storage_base_url, storage_account, extension_namespace, extension_name, extension_version, label, supported_os)
  <<~OPTIONS
    --management-url #{ENV['MANAGEMENT_URL']} `        # Azure Management URL for a non-public Azure cloud [$MANAGEMENT_URL] `\
    --subscription-id #{ENV['SUBSCRIPTION_ID']} `      # Subscription ID for the publisher subscription [$SUBSCRIPTION_ID] `\
    --subscription-cert #{ENV['SUBSCRIPTION_CERT']} `  # Path of subscription management certificate (.pem) file [$SUBSCRIPTION_CERT] `\
    --package #{env_package} `                         # Path of extension package (.zip) `\
    --storage-base-url #{storage_base_url} `           # Azure Storage base URL [$STORAGE_BASE_URL] `\
    --storage-account #{storage_account} `             # Name of an existing storage account to be used in uploading the extension package temporarily `\
    --namespace #{extension_namespace} `                            # Publisher namespace e.g. Microsoft.Azure.Extensions [$EXTENSION_NAMESPACE] `\
    --name #{extension_name} `                                      # Name of the extension e.g. FooExtension [$EXTENSION_NAME] `\
    --version #{extension_version} `                                # Version of the extension package e.g. 1.0.0 `\
    --label  'Chef Extension for #{label}' `                        # Human readable name of the extension `\
    --description 'Chef Extension that sets up chef-client on VM' ` # Description of the extension `\
    --eula-url #{CHEF_URL} `                           # URL to the End-User License Agreement page `\
    --privacy-url #{CHEF_URL} `                        # URL to the Privacy Policy page `\
    --homepage-url #{CHEF_URL} `                       # URL to the homepage of the extension `\
    --company 'Chef Software, Inc.' `                  # Human-readable Company Name of the publisher `\
    --supported-os #{supported_os} `                   # Extension platform e.g. 'Linux' `
  OPTIONS
end

# Publishes a new type of extension internally.
def new_extension_version_opts(manifest_path)
  <<-OPTIONS
    --management-url #{ENV['MANAGEMENT_URL']} `         # Azure Management URL for a non-public Azure cloud [$MANAGEMENT_URL] `\
    --subscription-id  #{ENV['SUBSCRIPTION_ID']} `      # Subscription ID for the publisher subscription [$SUBSCRIPTION_ID] `\
    --subscription-cert  #{ENV['SUBSCRIPTION_CERT']} `  # Path of subscription management certificate (.pem) file [$SUBSCRIPTION_CERT] `\
    --manifest #{manifest_path} `                       # Path of extension manifest file (XML output of 'new-extension-manifest') `
  OPTIONS
end

# Promote published internal extension to PROD in one or more locations.
def promote_opts(manifest_path, regions)
  <<-OPTIONS
    --management-url #{ENV['MANAGEMENT_URL']} `         # Azure Management URL for a non-public Azure cloud [$MANAGEMENT_URL]`\
    --subscription-id  #{ENV['SUBSCRIPTION_ID']} `      # Subscription ID for the publisher subscription [$SUBSCRIPTION_ID]`\
    --subscription-cert  #{ENV['SUBSCRIPTION_CERT']} `  # Path of subscription management certificate (.pem) file [$SUBSCRIPTION_CERT]`\
    --manifest #{manifest_path} `                       # Path of extension manifest file (XML output of 'new-extension-manifest')`\
    --region  #{regions.map { |reg| "--region \"#{reg}\" " }.join} ` # List of one or more regions to rollout an extension (e.g. 'Japan East')`
  OPTIONS
end

# Promote published extension to all Locations.
# This might be similar to new_extension_version_opts and possibly DRYied up.
# Intentionally keeping it as is right now for a better documentation.
def promote_all_regions_opts(manifest_path)
  <<-OPTIONS
    --management-url #{ENV['MANAGEMENT_URL']} `         # Azure Management URL for a non-public Azure cloud [$MANAGEMENT_URL] `\
    --subscription-id  #{ENV['SUBSCRIPTION_ID']} `      # Subscription ID for the publisher subscription [$SUBSCRIPTION_ID] `\
    --subscription-cert  #{ENV['SUBSCRIPTION_CERT']} `  # Path of subscription management certificate (.pem) file [$SUBSCRIPTION_CERT] `\
    --manifest #{manifest_path} `                       # Path of extension manifest file (XML output of 'new-extension-manifest')`
  OPTIONS
end

# Marks the specified version of the extension internal. Does not delete.
def unpublish_version_opts(extension_namespace, extension_name, extension_version)
  <<-OPTIONS
    --management-url #{ENV['MANAGEMENT_URL']} `         # Azure Management URL for a non-public Azure cloud [$MANAGEMENT_URL] `\
    --subscription-id  #{ENV['SUBSCRIPTION_ID']} `      # Subscription ID for the publisher subscription [$SUBSCRIPTION_ID] `\
    --subscription-cert  #{ENV['SUBSCRIPTION_CERT']} `  # Path of subscription management certificate (.pem) file [$SUBSCRIPTION_CERT] `\
    --namespace #{extension_namespace} `                # Publisher namespace e.g. Microsoft.Azure.Extensions [$EXTENSION_NAMESPACE] `\
    --name #{extension_name} `                          # Name of the extension e.g. FooExtension [$EXTENSION_NAME] `\
    --version #{extension_version} `                    # Version of the extension package e.g. 1.0.0 `
  OPTIONS
end

# Deletes the extension version. It should be unpublished first.
# This might be similar to unpublish_version_opts and possibly DRYied up.
# Intentionally keeping it as is right now for a better documentation.
def delete_version_opts(extension_namespace, extension_name, extension_version)
  <<-OPTIONS
    --management-url #{ENV['MANAGEMENT_URL']} `         # Azure Management URL for a non-public Azure cloud [$MANAGEMENT_URL] `\
    --subscription-id  #{ENV['SUBSCRIPTION_ID']} `      # Subscription ID for the publisher subscription [$SUBSCRIPTION_ID] `\
    --subscription-cert  #{ENV['SUBSCRIPTION_CERT']} `  # Path of subscription management certificate (.pem) file [$SUBSCRIPTION_CERT] `\
    --namespace #{extension_namespace} `                # Publisher namespace e.g. Microsoft.Azure.Extensions [$EXTENSION_NAMESPACE] `\
    --name #{extension_name} `                          # Name of the extension e.g. FooExtension [$EXTENSION_NAME] `\
    --version #{extension_version} `                    # Version of the extension package e.g. 1.0.0 `
  OPTIONS
end

###############################################################################
# Methods developed by using meta-programming to invoke azure-extensions-cli
# commands with their respective options.
# Example of a generated command:
# run_promote_all_regions("PATH TO XML") =>
# azure-extensions-cli_windows_amd64.exe promote-all-regions
#    --management-url URL
#    --subscription-id SUB_ID
#    --subscription-cert SUB_CERT
#    --manifest PATH TO XML
###############################################################################

%w{new_extension_manifest new_extension_version promote promote_all_regions
 unpublish_version delete_version}.each do |method|
  define_method "run_#{method}" do |*args|
    cli_command = method.tr("_", "-")
    begin
      options = send(:"#{method}_opts", *args)
      shell_command = "#{ENV['azure_extension_cli']} #{cli_command} #{options}"
      puts "Executing: #{shell_command}"
      cli_cmd = Mixlib::ShellOut.new(shell_command)
      result = cli_cmd.run_command
      result.error!
      puts "#{cli_command} has been executed successfully !"
      return result
    rescue Mixlib::ShellOut::ShellCommandFailed => e
      error_and_exit!("Failure while running `#{ENV['azure_extension_cli']} #{cli_command}`: #{e}")
    end
  end
end

###############################################################################
# Helper methods to verify provided inputs and building cli related commands
###############################################################################

def mgmt_uri(deploy_type)
  case deploy_type
  when /(^#{PRODUCTION}$|^#{DELETE_FROM_PRODUCTION}$)/
    "https://management.core.windows.net/"
  when /(^#{GOV}$|^#{DELETE_FROM_GOV}$)/
    "https://management.core.usgovcloudapi.net/"
  when /(^#{PREVIEW}$|^#{DELETE_FROM_PREVIEW}$)/
    "https://management-preview.core.windows-int.net/"
  end
end

def zip_folder_name(version, platform, date_tag = Date.today.strftime("%Y%m%d"))
  "#{PACKAGE_NAME}_#{version}_#{date_tag}_#{platform}.zip"
end

def xml_file_name(version, platform, date_tag = Date.today.strftime("%Y%m%d"))
  "#{MANIFEST_NAME}_#{version}_#{date_tag}_#{platform}"
end

def assert_git_state
  is_crlf = `git config --global core.autocrlf`
  unless is_crlf.chomp == "false"
    error_and_exit! "Please set the git crlf setting and clone, so git does not \
    auto convert newlines to crlf. \
    [EX: git config --global core.autocrlf false]"
  end
end

def assert_env_vars
  ENVIRONMENT_VARIABLES.each do |key, value|
    error_and_exit! "Please set the environment variable - \"#{key}\" for [#{value}]" unless ENV[key]
  end
end

# sets the common environment varaibles for Chef Extension
def set_env_vars(subscription_id, deploy_type)
  env_vars = {
    "SUBSCRIPTION_ID" => subscription_id,
    "MANAGEMENT_URL" => mgmt_uri(deploy_type),
    "EXTENSION_NAMESPACE" => "Chef.Bootstrap.WindowsAzure",
  }

  env_vars.each do |var, value|
    ENV[var] = value
  end
end

def assert_publish_env_vars
  PUBLISH_ENV_VARS.each do |key, value|
    error_and_exit! "Please set the environment variable - \"#{key}\" for [#{value}]" unless ENV[key]
  end
end

def assert_deploy_type_params(deploy_type)
  unless DEPLOY_TYPE.include?(deploy_type)
    error_and_exit! "Invalid parameter 'deploy_type'. \
    Valid values are: \"#{DEPLOY_TYPE.join('", "')}\""
  end
end

def assert_delete_type_params(delete_type)
  error_and_exit! "This task is supported on for deploy_types: \"#{DELETE_FROM_GOV}\" and \"#{DELETE_FROM_PRODUCTION}\"" unless DELETE_TYPE.include?(delete_type)
end

def assert_operation_params(operation)
  error_and_exit! 'operation parameter should be either "new" or "update"' unless %w{new update}.include?(operation)
end

def assert_internal_or_public_params(internal_or_public)
  unless INTERNAL_OR_PUBLIC.include?(internal_or_public)
    error_and_exit! "Invalid parameter 'internal_or_public'. \
    Valid values are: \"#{INTERNAL_OR_PUBLIC.join('", "')}\""
  end
end

def assert_gov_regions(regions)
  unless (invalid_regions = (regions - GOV_REGIONS)).empty?
    error_and_exit! "Invalid Region: #{invalid_regions.join(', ')}. \
    Valid regions for GOV Cloud are: \"#{GOV_REGIONS.join('", "')}\""
  end
end

def assert_chef_deploy_namespace_params(chef_deploy_namespace)
  error_and_exit! "'chef_deploy_namespace' must be specified." unless chef_deploy_namespace
end

def assert_extension_version(version)
  error_and_exit! "'extension_version' must be specified." unless version
end

def assert_build_date(build_date_yyyymmdd)
  error_and_exit! "Please specify the :build_date_yyyymmdd param used to identify the published build" unless build_date_yyyymmdd
end

def setup_sandbox
  FileUtils.mkdir_p CHEF_BUILD_DIR
  FileUtils.mkdir_p "#{CHEF_BUILD_DIR}/bin"
  FileUtils.mkdir_p "#{CHEF_BUILD_DIR}/gems"
end

def load_publish_settings
  assert_publish_env_vars
  doc = Nokogiri::XML(File.open(ENV["publishsettings"]))
  subscription_id = doc.at_css("Subscription").attribute("Id").value
  subscription_name = doc.at_css("Subscription").attribute("Name").value
  [subscription_id, subscription_name]
end

# Updates IsInternal as False for public release and True for internal release
def update_definition_xml(xml, args)
  doc = Nokogiri::XML(File.open(xml))
  internal = doc.at_css("IsInternalExtension")
  internal.content = is_internal?(args)
  File.write(xml, doc.to_xml)
end

def publish_uri(deploy_type, subscription_id, operation)
  uri = mgmt_uri(deploy_type) + "#{subscription_id}/services/extensions"
  uri += "?action=update" if operation == "update"
  uri
end

def load_publish_properties(platform)
  publish_options = JSON.parse(File.read("Publish.json"))

  definition_params = publish_options[platform]["definitionParams"]
  storage_account = definition_params["storageAccount"]
  storage_container = definition_params["storageContainer"]
  extension_name = definition_params["extensionName"]
  [storage_account, storage_container, extension_name]
end

def definition_xml(deploy_type, version, platform, chef_deploy_namespace)
  extension_zip_package = zip_folder_name(version, platform)
  storage_account, _storage_container, extension_name = load_publish_properties(platform)
  supported_os = platform == "windows" ? "windows" : "linux"

  storage_base_url = deploy_type == GOV ? "core.usgovcloudapi.net" : "core.windows.net"

  new_extension_manifest_args = [extension_zip_package, storage_base_url, storage_account,
    chef_deploy_namespace, extension_name, version, platform, supported_os]

  result = run_new_extension_manifest(*new_extension_manifest_args)
  definition_xml = result.stdout
  definition_xml
end

###############################################################################
# Rake tasks providing an efficient way of executing azure-extensions-cli
###############################################################################

desc "Cleans up the package sandbox"
task :clean do
  puts "Cleaning Chef Package..."
  FileUtils.rm_f(Dir.glob("*.zip"))
  puts "Deleting #{CHEF_BUILD_DIR} and #{PESTER_SANDBOX}"
  FileUtils.rm_rf(Dir.glob(CHEF_BUILD_DIR.to_s))
  FileUtils.rm_rf(Dir.glob(PESTER_SANDBOX.to_s))
  puts "Deleting gem file..."
  FileUtils.rm_f(Dir.glob("*.gem"))
  FileUtils.rm_f(Dir.glob("publishDefinitionXml_*"))
end

desc "Builds a azure chef extension gem."
task gem: [:clean] do
  puts "Building gem file..."
  puts `gem build *.gemspec`
end

desc "Builds the azure chef extension package Ex: build[platform, extension_version], default is build[windows]."
task :build, %i{target_type extension_version confirmation_required} => [:gem] do |_t, args|
  args.with_defaults(
    target_type: "windows",
    extension_version: EXTENSION_VERSION,
    confirmation_required: "true"
    )

  puts "Build called with args:
  target_type: #{args.target_type}
  extension_version: #{args.extension_version}
  confirmation_required: #{args.confirmation_required}"

  assert_git_state

  # Get user confirmation if we are downloading correct version.
  confirm!("build") if args.confirmation_required == "true"

  puts "Building #{args.target_type} package..."
  setup_sandbox

  # Copy platform specific files to package dir
  puts "Copying #{args.target_type} scripts to package directory..."
  package_list = args.target_type == "windows" ? WINDOWS_PACKAGE_LIST : LINUX_PACKAGE_LIST
  package_list.each do |src, dest|
    puts "Copy: src [#{src}] => dest [#{dest}]"
    if File.directory?(dest)
      FileUtils.cp_r Dir.glob(src), dest
    else
      FileUtils.cp Dir.glob(src).first, dest
    end
  end

  date_tag = Date.today.strftime("%Y%m%d")

  # Write a release tag file to zip. This will help during testing
  # to check if package was synced in PIR.
  FileUtils.touch "#{CHEF_BUILD_DIR}/version_#{args.extension_version}_#{date_tag}_#{args.target_type}"

  puts "\nCreating a zip package... \
  #{zip_folder_name(args.extension_version, args.target_type, date_tag)}" + "\n\n"
  Zip::File.open(zip_folder_name(args.extension_version, args.target_type, date_tag), Zip::File::CREATE) do |zipfile|
    Dir[File.join("#{CHEF_BUILD_DIR}/", "**", "**")].each do |file|
      zipfile.add(file.sub("#{CHEF_BUILD_DIR}/", ""), file)
    end
  end
end

desc "Publishes the azure chef extension package using publish.json Ex: publish[deploy_type, platform, extension_version], default is build[preview, windows]."
task :publish, %i{deploy_type target_type extension_version chef_deploy_namespace operation internal_or_public confirmation_required} do |_t, args|
  args.with_defaults(
    deploy_type: PREVIEW,
    target_type: "windows",
    extension_version: EXTENSION_VERSION,
    chef_deploy_namespace: "Chef.Bootstrap.WindowsAzure.Test",
    operation: "new",
    internal_or_public: CONFIRM_INTERNAL,
    confirmation_required: "true"
    )

  Rake::Task["build"].invoke(args[:target_type], args[:extension_version], args[:confirmation_required])

  puts "Publish called with args:
  deploy_type: #{args.deploy_type}
  target_type: #{args.target_type}
  extension_version: #{args.extension_version}
  chef_deploy_namespace: #{args.chef_deploy_namespace}
  operation: #{args.operation}
  internal_or_public: #{args.internal_or_public}
  confirmation_required: #{args.confirmation_required}"

  assert_deploy_type_params(args.deploy_type)
  assert_operation_params(args.operation)
  assert_internal_or_public_params(args.internal_or_public)

  subscription_id, subscription_name = load_publish_settings
  set_env_vars(subscription_id, args.deploy_type)
  assert_env_vars

  publish_uri = publish_uri(args.deploy_type, subscription_id, args.operation)

  definition_xml = definition_xml(args.deploy_type, args.extension_version, args.target_type, args.chef_deploy_namespace)

  puts <<~CONFIRMATION

    *****************************************
    This task creates a chef extension package and publishes to Azure #{args.deploy_type}.
    Details:
    -------
    Publish To:  ** #{args.deploy_type.gsub(/deploy_to_/, '')} **
    Subscription Name:  #{subscription_name}
    Extension Version:  #{args.extension_version}
    Publish Uri:  #{publish_uri}
    Build branch:  #{`git rev-parse --abbrev-ref HEAD`}
    Type:  #{args.internal_or_public == CONFIRM_INTERNAL ? 'Internal build' : 'Public release'}
    ****************************************
  CONFIRMATION

  # Get user confirmation, since we are publishing a new build to Azure.
  confirm!("Publish") if args.confirmation_required == "true"

  puts "Continuing with publish request..."

  date_tag = Date.today.strftime("%Y%m%d")
  manifest_file = File.new(xml_file_name(args.extension_version, args.target_type, date_tag), "w")
  xml_file_path = manifest_file.path
  puts "Writing publishDefinitionXml to #{xml_file_path}..."
  puts "[[\n#{definition_xml}\n]]"
  manifest_file.write(definition_xml)
  manifest_file.close
  new_extension_version_args = [xml_file_path]
  run_new_extension_version(*new_extension_version_args)
end

desc 'Promotes the extension in multiple regions for GOV Cloud. Provide semi-colon separated list of regions. Ex: "West Central US; North Central US; West US"'
task :promote, [:deploy_type, :target_type, :extension_version, :build_date_yyyymmdd, :regions, :confirmation_required] do |_t, args|
  args.with_defaults(
    deploy_type: PRODUCTION,
    target_type: "windows",
    extension_version: EXTENSION_VERSION,
    build_date_yyyymmdd: nil,
    regions: "USGov Virginia; USGov Iowa",
    confirmation_required: "true"
    )

  puts "Promote called with args:
  deploy_type: #{args.deploy_type}
  target_type: #{args.target_type}
  extension_version: #{args.extension_version}
  build_date_yyyymmdd: #{args.build_date_yyyymmdd}
  regions: #{args.regions}
  confirmation_required: #{args.confirmation_required}"

  regions = args.regions.split(";").strip

  assert_deploy_type_params(args.deploy_type)
  assert_gov_regions(regions) if args.deploy_type == GOV
  assert_build_date(args.build_date_yyyymmdd)
  subscription_id, subscription_name = load_publish_settings
  set_env_vars(subscription_id, args.deploy_type)
  assert_env_vars

  manifest_file = xml_file_name(args.extension_version, args.target_type, args.build_date_yyyymmdd)

  puts <<~CONFIRMATION

    *****************************************
    This task promotes the chef extension package to "#{regions.join('", "')}" regions.
    Details:
    -------
    Publish To:  ** #{args.deploy_type.gsub(/deploy_to_/, '')} **
    Subscription Name:  #{subscription_name}
    Extension Version:  #{args.extension_version}
    Build Date: #{args.build_date_yyyymmdd}
    #{regions.map.with_index { |reg, i| "Region #{i + 1}: #{reg}" }.join("\n    ")}
    ****************************************
  CONFIRMATION

  # Get user confirmation, since we are publishing a new build to Azure.
  confirm!("Promote") if args.confirmation_required == "true"

  puts "Promoting the extension to \"#{regions.join('", "')}\" regions..."
  promote_args = [manifest_file, regions]
  run_promote(*promote_args)
end

desc "Unpublishes the azure chef extension package which was publised in some Regions."
task :unpublish_version, [:deploy_type, :target_type, :extension_version, :confirmation_required] do |_t, args|
  args.with_defaults(
    deploy_type: DELETE_FROM_PRODUCTION,
    target_type: "windows",
    extension_version: nil,
    confirmation_required: "true"
    )

  puts "Unpublish Version called with args:
  deploy_type: #{args.deploy_type}
  target_type: #{args.target_type}
  extension_version: #{args.extension_version}
  confirmation_required: #{args.confirmation_required}"

  assert_delete_type_params(args.deploy_type)
  subscription_id, subscription_name = load_publish_settings
  set_env_vars(subscription_id, args.deploy_type)
  assert_env_vars

  publish_options = JSON.parse(File.read("Publish.json"))
  extension_name = publish_options[args.target_type]["definitionParams"]["extensionName"]

  # Get user confirmation, since we are deleting from Azure.
  puts <<~CONFIRMATION

    *****************************************
    This task unpublishes a published chef extension package from Azure #{args.deploy_type}.
    Details:
    -------
    Unpublish from:  ** #{args.deploy_type.gsub(/delete_from_/, '')} **
    Subscription Name:  #{subscription_name}
    Publisher Name:     #{ENV['EXTENSION_NAMESPACE']}
    Extension Name:     #{extension_name}
    ****************************************
  CONFIRMATION

  confirm!("Unpublish") if args.confirmation_required == "true"

  puts "Continuing with unpublish request..."
  unpublish_version_args = [ENV["EXTENSION_NAMESPACE"], extension_name, args.extension_version]
  run_unpublish_version(*unpublish_version_args)
end

desc "Deletes the azure chef extension package which was publised as internal Ex: publish[deploy_type, platform, extension_version], default is build[preview,windows]."
task :delete, [:deploy_type, :target_type, :chef_deploy_namespace, :extension_version, :confirmation_required] do |_t, args|
  args.with_defaults(
    deploy_type: DELETE_FROM_PREVIEW,
    target_type: "windows",
    chef_deploy_namespace: nil,
    extension_version: nil,
    confirmation_required: "true"
    )

  puts "Unpublish Version called with args:
  deploy_type: #{args.deploy_type}
  target_type: #{args.target_type}
  chef_deploy_namespace: #{args.chef_deploy_namespace}
  extension_version: #{args.extension_version}
  confirmation_required: #{args.confirmation_required}"

  assert_delete_type_params(args.deploy_type)
  assert_chef_deploy_namespace_params(args.chef_deploy_namespace)
  assert_extension_version(args.extension_version)

  subscription_id, subscription_name = load_publish_settings
  set_env_vars(subscription_id, args.deploy_type)
  assert_env_vars

  publish_options = JSON.parse(File.read("Publish.json"))
  extension_name = publish_options[args.target_type]["definitionParams"]["extensionName"]

  # Get user confirmation, since we are deleting from Azure.
  puts <<~CONFIRMATION

    *****************************************
    This task unpublishes a published chef extension package from Azure #{args.deploy_type}.
    Details:
    -------
    Delete from:     ** #{args.deploy_type.gsub(/delete_from_/, '')} **
    Subscription Name:  #{subscription_name}
    Publisher Name:     #{ENV['EXTENSION_NAMESPACE']}
    Extension Name:     #{extension_name}
    ****************************************
  CONFIRMATION

  confirm!("Delete") if args.confirmation_required == "true"

  puts "Continuing with delete request..."
  delete_version_args = [ENV["EXTENSION_NAMESPACE"], extension_name, args.extension_version]
  run_delete_version(*delete_version_args)
end

desc 'Updates the azure chef extension package metadata which was publised Ex: update["definitionxml.xml"].'
task :update, [:deploy_type, :target_type, :extension_version, :build_date_yyyymmdd, :chef_deploy_namespace, :internal_or_public, :confirmation_required] do |_t, args|
  args.with_defaults(
    deploy_type: PREVIEW,
    target_type: "windows",
    extension_version: EXTENSION_VERSION,
    build_date_yyyymmdd: nil,
    chef_deploy_namespace: "Chef.Bootstrap.WindowsAzure.Test",
    internal_or_public: CONFIRM_INTERNAL,
    confirmation_required: "true"
    )

  puts "Update Version called with args:
  deploy_type: #{args.deploy_type}
  target_type: #{args.target_type}
  extension_version: #{args.extension_version}
  build_date_yyyymmdd: #{args.build_date_yyyymmdd}
  chef_deploy_namespace: #{args.chef_deploy_namespace}
  internal_or_public: #{args.internal_or_public}
  confirmation_required: #{args.confirmation_required}"

  assert_deploy_params(args.deploy_type, args.internal_or_public)
  assert_build_date(build_date_yyyymmdd)

  manifest_file = xml_file_name(args.extension_version, args.target_type, args.build_date_yyyymmdd)
  update_definition_xml(manifest_file, args) # Updates IsInternal as False for public release and True for internal release

  subscription_id, subscription_name = load_publish_settings
  set_env_vars(subscription_id, args.deploy_type)
  assert_env_vars

  publish_uri = publish_uri(args.deploy_type, subscription_id, "update")

  puts <<~CONFIRMATION

    *****************************************
    This task updates the chef extension package which is already published to Azure #{args.deploy_type}.
    Details:
    -------
    Publish To:  ** #{args.deploy_type.gsub(/deploy_to_/, '')} **
    Subscription Name:  #{subscription_name}
    Extension Version:  #{args.extension_version}
    Build Date: #{args.build_date_yyyymmdd}
    Publish Uri:  #{publish_uri}
    Type:  #{is_internal?(args) ? 'Internal build' : 'Public release'}
    ****************************************
  CONFIRMATION

  # Get user confirmation, since we are publishing a new build to Azure.
  confirm!("Update") if args.confirmation_required == "true"

  puts "Continuing with udpate request..."

  promote_all_regions_args = [manifest_file]
  run_promote_all_regions(*promote_all_regions_args)
end

task :init_pester do
  puts "Initializing Pester to run powershell unit tests..."
  puts `powershell -Command if (Test-Path "#{PESTER_SANDBOX}") {Remove-Item -Recurse -Force #{PESTER_SANDBOX}"}`
  puts `powershell "mkdir #{PESTER_SANDBOX}; cd #{PESTER_SANDBOX}; git clone --branch #{PESTER_VER_TAG} \'#{PESTER_GIT_URL}\'"; cd ..`
end

# Its runs pester unit tests
# have a winspec task that can be used to trigger tests in jenkins
desc 'Runs pester unit tests ex: rake spec["spec\\ps_specs\\sample.Tests.ps1"]'
task :winspec, [:spec_path] => [:init_pester] do |_t, args|
  puts "\nRunning unit tests for powershell scripts..."
  # Default: runs all tests under spec dir,
  # user can specify individual test file
  # Ex: rake spec["spec\ps_specs\sample.Tests.ps1"]
  args.with_defaults(spec_path: "spec/ps_specs")

  # run pester tests
  puts `powershell -ExecutionPolicy Unrestricted Import-Module #{PESTER_SANDBOX}/Pester/Pester.psm1; Invoke-Pester -relative_path #{args.spec_path}`
end

# rspec
begin
  require "rspec/core/rake_task"
  desc "Run all specs in spec directory"
  RSpec::Core::RakeTask.new(:spec) do |t|
    t.rspec_opts = ["--format", "documentation"]
    t.pattern = "spec/**/**/*_spec.rb"
  end
rescue LoadError
  STDERR.puts "\n*** RSpec not available. (sudo) gem install rspec to run unit tests. ***\n\n"
end
