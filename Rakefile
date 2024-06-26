require 'facterdb/version'

begin
  require 'rspec/core/rake_task'
  RSpec::Core::RakeTask.new(:spec)
rescue LoadError
end

begin
  require 'github_changelog_generator/task'
  GitHubChangelogGenerator::RakeTask.new :changelog do |config|
    config.future_release = FacterDB::Version::STRING
    config.exclude_labels = %w[duplicate question invalid wontfix wont-fix maintenance skip-changelog github_actions]
    config.user = 'voxpupuli'
    config.project = 'facterdb'
    config.release_url = 'https://rubygems.org/gems/facterdb/versions/%s'
    config.since_tag = '0.3.11'
  end
rescue LoadError
  desc 'Install github_changelog_generator to get access to automatic changelog generation'
  task :changelog do
    raise 'Install github_changelog_generator to get access to automatic changelog generation'
  end
end

begin
  require 'yard'
rescue LoadError
  # No yard
else
  YARD::Rake::YardocTask.new
  task yard: :database # rubocop:disable Rake/Desc
end

# Generate a human-friendly OS label based on a given factset
def factset_to_os_label(fs)
  os_rel  = '???'
  os_name = '????'
  if fs.key?(:os) && fs[:os].key?('release') && (fs[:os]['release']['major'] =~ /\d/)
    os_name = fs[:os]['name']
    fail    = {}
    os      = fs[:os]
    distro = os.fetch('lsb', os.fetch('distro', fail))
    os_id = if distro.key?('id')
              distro['id']
            elsif distro.key?('distid')
              distro['distid']
            else
              '@@@@@@@@@@'
            end
    os_rel  = fs[:os]['release']['major']
    os__rel = fs[:os]['release']['full']
  elsif fs.key? :operatingsystem
    os_name  = fs[:operatingsystem]
    os_id    = fs.fetch(:lsbdistid, '@@@@@@@@@@')
    os_rel   = fs.fetch(:operatingsystemmajrelease, fs.fetch(:lsbmajdistrelease, nil))
    os__rel  = fs.fetch(:lsbdistrelease, fs.fetch(:operatingsystemrelease, '@@@'))
  else
    pp fs
    raise('ERROR: unrecognized facterset format')
  end
  # Sanitize OS names to match the formats used in the facterdb README
  label = "#{os_name} #{os_rel}"
  if /^(Archlinux|Gentoo)$/.match?(os_name)
    label = os_name
  elsif os_name =~ /^(SLES|FreeBSD|OpenSuSE)$/ ||
        (os_name =~ /^(RedHat|Scientific|OracleLinux|CentOS|AlmaLinux)/ && os_rel.nil?)
    label = "#{os_name} #{os__rel.split('.').first}"
  elsif /^(OpenBSD|Ubuntu|Fedora)$/.match?(os_name)
    label = "#{os_name} #{os__rel}"
  elsif /^(Solaris)/.match?(os_name)
    label = if fs.key?(:os) && fs[:os].key?('release') && (fs[:os]['release']['major'] =~ /\d/)
              "#{os_name} #{os_rel}"
            elsif fs.key?(:operatingsystemmajrelease)
              "#{os_name} #{os_rel}"
            else
              "#{os_name} #{os__rel.split('.')[1]}"
            end
  elsif /sles-15-/.match?(fs[:_facterdb_filename])
    label = 'SLES 15'
  elsif os_name.start_with?('Debian') && os_id == 'LinuxMint'
    label = "#{os_id} #{fs[:lsbmajdistrelease]}"
  elsif os_name.start_with?('Debian') && os_id == 'Raspbian'
    label = "#{os_id} #{os_rel}"
  elsif /^windows$/.match?(os_name)
    db_filename = fs[:_facterdb_filename] || 'there_is_no_filename'
    label = if /windows-10-/.match?(db_filename)
              'Windows 10'
            elsif /windows-11-/.match?(db_filename)
              'Windows 11'
            elsif /windows-8[\d.]*-/.match?(db_filename)
              "Windows #{os__rel.sub('6.2.9200', '8').sub('6.3.9600', '8.1')}"
            elsif /windows-.+-core-/.match?(db_filename)
              "Windows Server #{os__rel.sub('6.3.9600', '2012 R2')} Core"
            elsif db_filename =~ /windows-2008/ || db_filename =~ /windows-2012/ || db_filename =~ /windows-2016/
              "Windows Server #{os__rel.sub('6.1.7600', '2008 R2').sub('6.3.9600', '2012 R2').sub('10.0.14393',
                                                                                                  '2016')}"
            elsif /windows-2019/.match?(db_filename)
              'Windows Server 2019'
            elsif /windows-2022/.match?(db_filename)
              'Windows Server 2022'
            else
              "#{os_name} #{os__rel}"
            end
  end

  label
end

def normalize_arch(arch_str)
  case arch_str
  when 'x86_64', 'amd64'
    'x86_64'
  when 'i386'
    'i386'
  else
    arch_str
  end
end

def factset_hash_to_markdown(h, caption1: nil, caption2: nil, caption3: nil)
  content = ''

  h.each do |label1, data1|
    indent = ''
    content += "#{indent} - "
    content += "#{caption1} " if caption1
    content += "#{label1}\n"
    indent += '  '
    data1.each do |label2, data2|
      content += "#{indent}- "
      content += "#{caption2} " if caption2
      content += "#{label2}:"

      data2.each do |label3|
        content += " #{caption3}" if caption3
        content += " #{label3}"
      end
      content += "\n"
    end
  end

  content
end

desc 'generate a markdown table of Facter/OS coverage'
task :database do
  require_relative 'lib/facterdb'
  FileUtils.mkdir_p 'database'

  # Turn on the source injection
  old_env = ENV.fetch('FACTERDB_INJECT_SOURCE', nil)
  ENV['FACTERDB_INJECT_SOURCE'] = 'true'
  factsets = FacterDB.get_facts
  # Restore the source injection
  ENV['FACTERDB_INJECT_SOURCE'] = old_env

  facter_versions = factsets.map do |x|
    Gem::Version.new(x[:facterversion].split('.')[0..1].join('.'))
  end.uniq.sort.map(&:to_s)

  # Old table
  os_facter_matrix = {}

  # New lists
  os_facter_arch_list = {}
  os_arch_facter_list = {}
  facter_os_arch_list = {}
  # Can't think of any reason facter_arch_os_list would be useful
  arch_os_facter_list = {}
  # Can't think of any reason arch_facter_os_list would be useful

  # Process the facts and create a hash of all the OS and facter combinations
  factsets.each do |facts|
    fv = facts[:facterversion].split('.')[0..1].join('.')
    arch = normalize_arch(facts[:hardwaremodel]) || 'Missing Value'
    label = factset_to_os_label(facts)
    os_facter_matrix[label] ||= {}
    os_facter_matrix[label][fv] ||= 0
    os_facter_matrix[label][fv] += 1

    os_facter_arch_list[label] ||= {}
    os_facter_arch_list[label][fv] ||= []
    os_facter_arch_list[label][fv] << arch

    os_arch_facter_list[label] ||= {}
    os_arch_facter_list[label][arch] ||= []
    os_arch_facter_list[label][arch] << fv

    facter_os_arch_list[fv] ||= {}
    facter_os_arch_list[fv][label] ||= []
    facter_os_arch_list[fv][label] << arch

    arch_os_facter_list[arch] ||= {}
    arch_os_facter_list[arch][label] ||= []
    arch_os_facter_list[arch][label] << fv
  end
  # Extract the OS list
  os_versions = os_facter_matrix.keys.uniq.sort_by do |label|
    string_pieces = label.split(/\d+/)
    number_pieces = label.split(/\D+/).map(&:to_i)
    number_pieces.shift
    string_pieces.zip(number_pieces).flatten.compact
  end

  # Write out a nice table
  os_version_width = (os_versions.map { |x| x.size } + [17]).max
  facter_width = 3

  rows = [
    ['operating system'.center(os_version_width)] + facter_versions,
    ['-' * os_version_width] + (['-' * facter_width] * facter_versions.length),
  ]

  os_versions.each do |label|
    fvs = facter_versions.map do |facter_version|
      version = os_facter_matrix[label][facter_version] || 0
      (version > 0) ? version.to_s : ''
    end
    rows << ([label.ljust(os_version_width)] + fvs.map { |fv| fv.center(facter_width) })
  end

  content = "# @title Table of Available Factsets\n\n"
  content += "# Facter version and Operating System coverage\n\n"
  rows.each do |row|
    content += "| #{row.join(' | ')} |\n"
  end
  content += "\n\nWhere the number (1, 2 etc.) are the number of factsets for that OS and facter combination (e.g., x86_64 and i386 architectures).\n"

  File.write(File.join(__dir__, 'database', 'table.md'), content)

  content = "# @title Available Factsets Grouped By OS -> Facter -> Architecture\n\n"
  content += "# Available Facts Grouped By OS -> Facter\n\n"
  content += factset_hash_to_markdown(os_facter_arch_list, caption2: 'Facter')
  File.write(File.join(__dir__, 'database', 'list_os_facter_arch.md'), content)

  content = "# @title Available Factsets Grouped By OS -> Architecture -> Facter Version\n\n"
  content += "# Available Facts Grouped By OS -> Arch\n\n"
  content += factset_hash_to_markdown(os_arch_facter_list)
  File.write(File.join(__dir__, 'database', 'list_os_arch_facter.md'), content)

  content = "# @title Available Factsets Grouped By Facter Version -> OS -> Architecture\n\n"
  content += "# Available Facts Grouped By Facter -> OS\n\n"
  content += factset_hash_to_markdown(facter_os_arch_list, caption1: 'Facter')
  File.write(File.join(__dir__, 'database', 'list_facter_os_arch.md'), content)

  content = "# @title Available Factsets Grouped By Architecture -> OS -> Facter Version\n\n"
  content += "# Available Facts Grouped By Arch -> OS\n\n"
  content += factset_hash_to_markdown(arch_os_facter_list)
  File.write(File.join(__dir__, 'database', 'list_arch_os_facter.md'), content)
end

begin
  require 'voxpupuli/rubocop/rake'
rescue LoadError
  # the voxpupuli-rubocop gem is optional
end

task default: 'spec'
