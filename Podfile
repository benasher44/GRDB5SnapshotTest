target 'GRDB5SnapshotTest' do
  use_frameworks!

  pod 'GRDB.swift'
  pod 'sqlite3'

end

# Does the replacement in a test file first to avoid dirtying a pod and triggering a rebuild
def build_safe_sed(pattern, file)
  test_file = "#{file}.test"
  system "sed -e '#{pattern}' '#{file}' > '#{test_file}'"
  FileUtils.cp(test_file, file) unless FileUtils.compare_file(file, test_file)
  File.delete test_file
end

# Patch GRDB Target to depend on our sqlite3
def patch_grdb_target(target, all_targets)
  # Replace system #includes with our @imports
  build_safe_sed 's/#include <sqlite3.h>/@import sqlite3;/g', "#{__dir__}/Pods/GRDB.swift/Support/grdb_config.h"

  # Uncomment below to namespace snapshot struct
  # Then, remove the Pods directory and re-run pod install
  #
  # build_safe_sed 's/typedef struct sqlite3_snapshot [{]/typedef struct grdb_sqlite3_snapshot {/g', "#{__dir__}/Pods/GRDB.swift/Support/grdb_config.h"
  # build_safe_sed 's/[}] sqlite3_snapshot;/} grdb_sqlite3_snapshot;/g', "#{__dir__}/Pods/GRDB.swift/Support/grdb_config.h"

  # Ensure sqlite3 is built before GRDB
  target.add_dependency(all_targets.find { |t| t.name == 'sqlite3' })
  target.build_configurations.each do |config|
    # Set the flag to skip importing system sqlite
    config.build_settings['OTHER_SWIFT_FLAGS'] ||= ['$(inherited)', '-D', 'GRDBCUSTOMSQLITE', '-D', 'SQLITE_ENABLE_FTS5']

    # Add sqlite3 to framework search paths
    config.build_settings['FRAMEWORK_SEARCH_PATHS'] ||= ['$(inherited)', '"${PODS_CONFIGURATION_BUILD_DIR}/sqlite3"']
  end
end

def patch_sqlite3_target(target)
  target.build_configurations.each do |config|
    # Set the custom sqlite build flags we want
    config.build_settings['GCC_PREPROCESSOR_DEFINITIONS'] ||= ['$(inherited)', 'SQLITE_MAX_VARIABLE_NUMBER=2147483000', 'SQLITE_ENABLE_FTS5=1', 'SQLITE_ENABLE_JSON1=1', 'SQLITE_USE_URI=1']
  end
end

post_install do |installer|
  # REMOVES SQLITE3 FROM PODS DEPENDENCIES
  Dir.glob("#{__dir__}/Pods/Target\\ Support\\ Files/*/*.xcconfig").each do |f|
    build_safe_sed 's/-l\"sqlite3\" //g', f
    build_safe_sed 's/-l\"sqlite3\.0\\\" //g', f
  end

  all_targets = installer.generated_projects.flat_map(&:targets)
  all_targets.each do |target|
    patch_sqlite3_target(target) if target.name == 'sqlite3'
    patch_grdb_target(target, all_targets) if target.name == 'GRDB.swift'
  end
end

