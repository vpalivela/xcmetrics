# Directories
BUILD_DIR    = File.expand_path('build')
REPORTS_DIR  = BUILD_DIR + "/reports-" + Time.now.strftime("%m-%d-%Y")

# Output
XCBUILD_LOG      = BUILD_DIR + "/xcodebuild.log"
XCCOV_LOG        = BUILD_DIR + "/xccoverage.log"
LINT_DESTINATION = REPORTS_DIR + "/lint.html"

# Libraries
OCLINT_BIN_DIR     = ENV["OCLINT_BIN_DIR"] || "ThirdParty/oclint-0.9.dev.648e9af/bin"
XCODECOVERAGE_DIR  = ENV["XCODECOVERAGE_DIR"] || "XcodeCoverage"
XCPRETTY_AVALIABLE = Gem::Specification::find_all_by_name('xcpretty').any?

# Build
WORKSPACE           = 'TitanIOS.xcworkspace'
DEFAULT_SCHEME      = 'TitanIOS'
SIMULATOR_NAME      = ENV["SIMULATOR_NAME"] || "iPhone 5"
SDK_BUILD_VERSION   = ENV["SDK_BUILD_VERSION"] || "8.1"
BUILD_CONFIGURATION = ENV["BUILD_CONFIGURATION"] || "Debug"
OCLINT_EXCLUDES     = [
  'ThirdParty', 'Crashlytics', 'Experitest', 'FacebookSDK',
  'Instruments', 'Lib', 'Pods', 'OpenUDID.m', 'Resources',
  'TestData', 'Frameworks', 'Supporting Files', 'Unit Tests',
  'Logic Tests', '\.storyboard$', '\.xib$', 'Contract Tests'
]

##############################################################################
# Standard tasks
##############################################################################

task :default do
  system "rake --tasks"
end

desc "Shows the tasks supported"
task :help do
  system "rake --tasks"
end

desc "All in one task to build, test, generate report and open them."
task :go => ['test', 'lint', 'cov', 'reports']

desc "Task for CI Box"
task :ci => ['test','lint','cov']

desc "Cleans the build artifacts"
task :clean do
  xcbuild('clean')

  run_cmd("rm -rf ~/Library/Developer/Xcode/DerivedData/ModuleCache/*", 'ModuleCache Cleanup')
  run_cmd("rm -rf ~/Library/Developer/Xcode/DerivedData/#{DEFAULT_SCHEME}-*", 'DerivedData Cleanup')
  run_cmd('rm -rf build')
end

desc "Builds the application"
task :build do
  xcbuild('build')
end

desc "Tests the application"
task :test => :clean do
  close_simulator
  begin
    xcbuild("clean test", "--test -r html --output '#{REPORTS_DIR}/tests.html'")
  ensure
    close_simulator
  end
end

desc "Runs lint on the application"
task :lint do
  lint("#{lint_excludes}")
end

desc "Generates code coverage report"
task :cov => :gencov do
  run_cmd("#{XCODECOVERAGE_DIR}/cicov", "cicov")
  puts "\nCode Coverage Finished, open #{REPORTS_DIR} to view results".green
end

task :gencov do
  run_cmd("xcodebuild \
              -workspace TitanIOS.xcworkspace \
              -scheme TitanIOS \
              -sdk iphonesimulator#{SDK_BUILD_VERSION} \
              -destination platform='iOS Simulator',OS=#{SDK_BUILD_VERSION},name='iPhone Retina (4-inch)' \
              -configuration #{BUILD_CONFIGURATION} \
              -showBuildSettings | \
          egrep '( BUILT_PRODUCTS_DIR)|(NATIVE_ARCH )|(OBJECT_FILE_DIR_normal)|(SRCROOT)|(OBJROOT)' \
          | tr -d ' ' | sed -e 's/^/export /' | sed -e 's/$/\"/' | sed 's/=/=\"/g' | sed 's/NATIVE_ARCH/CURRENT_ARCH/' \> #{XCODECOVERAGE_DIR}/env.sh",
          "Load Build Variables")
end

task :reports do
  run_cmd("open #{REPORTS_DIR}")
end

##############################################################################
# Build Methods
##############################################################################

def xcbuild(build_type = '', xcpretty_args = '')
  unless File.exists?(BUILD_DIR)
    Dir.mkdir(BUILD_DIR)
    Dir.mkdir(REPORTS_DIR)
  end

  # Default scheme
  scheme ||= DEFAULT_SCHEME

  xcodebuild = "xcodebuild \
                      -workspace #{WORKSPACE} \
                      -scheme #{scheme} \
                      -sdk iphonesimulator#{SDK_BUILD_VERSION} \
                      -destination platform='iOS Simulator',OS=#{SDK_BUILD_VERSION},name='#{SIMULATOR_NAME}' \
                      -configuration #{BUILD_CONFIGURATION} \
                      #{build_type} 2>&1 | tee #{XCBUILD_LOG} "

  if XCPRETTY_AVALIABLE
    run_cmd(xcodebuild +
            "2>&1 | xcpretty -c --no-utf #{xcpretty_args}; \
            exit ${PIPESTATUS[0]}",
            "xcodebuild " + build_type)
  else
    run_cmd(xcodebuild, "xcodebuild " + build_type)
  end
end

def lint(lint_args = '')
  log_info("Starting","lint " + Time.now.inspect)

  if !File.exists?(XCBUILD_LOG)
    log_error("xcodebuild.log not found in #{BUILD_DIR}. Please run `rake build`")
  end

  if !File.exists?(OCLINT_BIN_DIR)
    log_error("Could not locate oclint. Please make sure oclint is installed and set the 'OCLINT_BIN_DIR' environment variable.")
  end

  if @ci_task
    report_type = "-report-type=pmd -o #{REPORTS_DIR}/oclint.xml"
  else
    report_type = "-report-type=html -o #{REPORTS_DIR}/oclint.html"
  end

  run_cmd("#{OCLINT_BIN_DIR}/oclint-xcodebuild #{lint_excludes}\
                #{XCBUILD_LOG}", "oclint-xcodebuild")

  run_cmd("#{OCLINT_BIN_DIR}/oclint-json-compilation-database \
                #{lint_args} -- \
                -disable-rule=FeatureEnvy \
                #{report_type} \
                -max-priority-1=100 \
                -max-priority-2=1000 \
                -max-priority-3=4000 \
                -rc CYCLOMATIC_COMPLEXITY=7 \
                -rc LONG_LINE=200 \
                -rc LONG_VARIABLE_NAME=50",
          "oclint")

  run_cmd("rm -rf compile_commands.json", "cleanup " + Time.now.inspect)
  puts "\nLint Finished, open #{REPORTS_DIR} to view results".green
end

def close_simulator
  begin
    log_info("Closing","iPhone Simulator")
    sh `killall -m -KILL \"iPhone Simulator\"`
  rescue
    nil
  end
end

def lint_excludes
  "-e '#{OCLINT_EXCLUDES.join("' -e '")}'"
end

##############################################################################
# Private Methods
##############################################################################

private

def run_cmd( cmd, desc = nil)
  desc ||= cmd
  log_info("Running", desc)
  Dir.mkdir(BUILD_DIR) unless File.exists?(BUILD_DIR)
  unless system("#{cmd}")
    log_error(desc)
  end
end

def log_info(action, description)
  puts ">".yellow + " #{action}".bold + " #{description}"
end

def log_error(description)
  puts "[!] FAILED #{description}".red
  exit 1
end

class String
  def colorize(color_code)
    "\e[#{color_code}m#{self}\e[0m"
  end

  def red
    colorize(31)
  end

  def green
    colorize(32)
  end

  def yellow
    colorize(33)
  end

  def bold
    "\e[1m#{self}\e[22m"
  end
end
