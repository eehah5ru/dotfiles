# coding: utf-8
# Based on https://github.com/ryanb/dotfiles/

require "erb"
require "fileutils"
require "pstore"
require "rake"
require "shellwords"

IGNORE = %w[
  dotfiles.sublime-project
  dotfiles.sublime-workspace
  completions
  extras
  LICENSE
  Rakefile
  README.md
  todo.org
].freeze

task default: "install"

desc "Install packages and dotfiles"
#task install: %w[install:dotfiles install:completions install:packages]
task install: %w[install:files]

desc "Warn if git origin is newer"
task :check do
  next unless system("git fetch origin")
  next if `git diff HEAD origin/master`.strip.empty?
  log(:yellow, "warning Working copy is out of date; consider `git pull`")
end

namespace :install do
  # desc "Install homebrew, etc. packages"
  # task packages: :check do
  #   %w(brew defaults).each do |type|
  #     log(:blue, "executing bin/#{type}-install …")
  #     system("bin/#{type}-install")
  #   end
  # end

  desc "Install files into user’s home directory"
  task files: %i[ check] do
    always_replace = false

    def do_install(a_file)
      case a_file.status
      when :identical
        log(:green, "identical #{a_file}")
      when :missing
        a_file.link!
      when :different
        if always_replace
          a_file.replace!
        elsif (answer = ask(:red, "overwrite? #{a_file}"))
          always_replace = true if answer == :always
          a_file.replace!
        else
          log(:gray, "skipping #{a_file}")
        end
      end
    end

    Dotfile.each do |dotfile|
      do_install(dotfile)
    end

    Binfile.each do |binfile|
      do_install(binfile)
    end
  end

  # desc "Install bash completion scripts into /usr/local"
  # task :completions do
  #   FileUtils.mkdir_p "/usr/local/etc/bash_completion.d"
  #   Dir[File.expand_path("completions/*", __dir__)].each do |script|
  #     basename = File.basename(script)
  #     target = "/usr/local/etc/bash_completion.d/#{basename}"
  #     log(:blue, "linking completions/#{basename}")
  #     FileUtils.rm_rf(target)
  #     FileUtils.ln_s(script, target)
  #   end
  # end

  # desc "Symlink the Sublime Packages/User directory"
  # task :link_sublime do
  #   dot_sublime = File.expand_path("~/.sublime")
  #   user_packages = File.expand_path("~/Library/Application Support/Sublime Text 3/Packages/User")
  #   if !File.exist?(user_packages)
  #     log(:magenta, "mkdir Library/Application Support/Sublime Text 3/Packages/User")
  #     FileUtils.mkdir_p(user_packages)
  #   end
  #   if File.directory?(user_packages) && ! File.symlink?(user_packages)
  #     log(:magenta, "mkdir .sublime")
  #     FileUtils.mkdir_p(dot_sublime)
  #     log(:blue, "copy  Library/Application Support/Sublime Text 3/Packages/User/*")
  #     FileUtils.cp_r(Dir.glob(user_packages.shellescape + "/*"), dot_sublime)
  #     log(:magenta, "rm    Library/Application Support/Sublime Text 3/Packages/User")
  #     FileUtils.rm_rf(user_packages)
  #     log(:blue, "linking Library/Application Support/Sublime Text 3/Packages/User")
  #     FileUtils.ln_s(dot_sublime, user_packages)
  #   end
  # end
end

def log(color, message, options={})
  begin
    require "highline"
  rescue LoadError
  end

  first, rest = message.split(" ", 2)
  first = first.ljust(10)

  if defined?(HighLine::String)
    first = HighLine::String.new(first).public_send(color)
  end

  line = [first, rest].join(" ")

  if line.end_with?(" ")
    print(line)
  else
    puts(line)
  end
end

def ask(color, question)
  log(color, "#{question} [yNaq]? ")

  case $stdin.gets.chomp
  when "a"
    :always
  when "y"
    true
  when "q"
    exit
  else
    false
  end
end

class FileList
  def self.each
    @include_only = false unless @include_only
    @ignore = [] unless @ignore

    `git ls-files -z`.split("\x0").each do |file|
      next if file =~ %r{^\.|/\.} # exclude hidden files
      next if @ignore.include?(file.split("/").first)

      next if @include_only and (not @include_only.include?(file.split("/").first))

      yield(new(file))
    end
  end

  attr_reader :file

  def initialize(file)
    @file = file
  end

  def erb?
    file =~ /\.erb\z/i
  end

  def name
    raise "unimplemented"
  end

  def to_s
    name
  end

  def target
    File.expand_path("~/#{name}")
  end

  def status
    if File.identical?(file, target)
      :identical
    elsif File.exist?(target) || File.symlink?(target)
      :different
    else
      :missing
    end
  end

  def link!(delete_first: false)
    ensure_target_directory

    if erb?
      log(:yellow, "generating #{self}")
      contents = ERB.new(File.read(file)).result(binding)

      log(:blue, "writing #{self}")
      FileUtils.rm_rf(target) if delete_first
      File.open(target, "w") do |out|
        out << contents
      end
    else
      log(:blue, "linking #{self}")
      FileUtils.rm_rf(target) if delete_first
      FileUtils.ln_s(File.absolute_path(file), target)
    end
  end

  def replace!
    link!(delete_first: true)
  end

  def ensure_target_directory
    directory = File.dirname(target)
    return if File.directory?(directory)

    log(:magenta, "mkdir #{File.dirname(name)}")
    FileUtils.mkdir_p(directory)
  end

  def prompt(label)
    default = pstore.transaction { pstore[label] }
    print default ? "#{label} (#{default}): " : "#{label}: "
    $stdout.flush

    entry = $stdin.gets.chomp
    result = entry.empty? && default ? default : entry

    pstore.transaction { pstore[label] = result }
    result
  end

  def pstore
    @_pstore ||= PStore.new(pstore_path)
  end

  def pstore_path
    File.join(__dir__, ".db")
  end
end

class Dotfile < FileList
  @ignore = IGNORE + %w{bin}

  def name
    ".#{file.sub(/\.erb\z/i, "")}"
  end

end

class Binfile < FileList
  @include_only = %{bin}
  @ignore = IGNORE

  def name
    "#{file.sub(/\.erb\z/i, "")}"
  end
end
