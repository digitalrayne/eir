# Rake-based build system for Eir
# Downloads, verifies, and builds sources into a usable toolkit, which is then used to compile a minimal distro.
#
# Author:: James Hebden (james@ec0.io)
# Copyright:: Copyleft 2016 James Hebden
# License:: GPLv3

require 'erb'
require 'yaml'
require 'open-uri'
require 'progressbar'
require 'digest'
require 'zlib'
require 'xz'
require 'bzip2/ffi'
require 'colorize'
require 'archive/tar/minitar'

#Suppress XZ deprecation notices
XZ.disable_deprecation_notices = true

#Get list of packages
puts "Loading packages from packages/*.yml".green
@packages = Rake::FileList.new( "packages/*.yml" )

#Get package longname (e.g. package-x.yz)
def get_package_longname ( source )

  #get basename (just the filename part of the path)
  name = File.basename( source )
  #Remove anything after a dot followed by at least two ASCII characters.
  #This will strip any extensions we're interested in passing to this function,
  #whilst preserving version information in files.
  #We care about the version information, becuase the loose standard for 
  #tarballs is for the archive to be extracted into a folder name somethign
  #along the lines of <project name>-<version>
  name.sub(/\.[a-z][a-z].*/, '')


end

#Get package configuration from YML metadata
def get_package_data ( package )

  #Load data from YML
  package_data = YAML.load_file( package ) 

  #Get package name, basename and metadata
  package_data[ 'longname' ] = get_package_longname( package_data[ 'file' ] ) 

  package_data

end

#Packages with metadata
@package_metadata = Hash.new 
@packages.each do |package|
  @package_metadata[ package ] = get_package_data( package )
end

#Download file via HTTP
def get_package_file ( dest, uri )

  #Open destination file
  File.open( dest, "w" ) do |dest_file|

    progress_bar = nil

    #Write file from open-uri initiated download file handle, register callbacks for progress bar.
    dest_file.write open( uri,
      :content_length_proc => lambda { |size|
        if size && size > 0
          progress_bar = ProgressBar.new( "...", size )
          progress_bar.file_transfer_mode
        end
      },
      :progress_proc => lambda { |position|
          progress_bar.set position if progress_bar
      }
    ).read 
  end
end

#Return hash of downloaded file, as string
def hash_package_file ( dest )

  "#{ Digest::SHA256.file( dest ) }"

end

#Open a decompression stream/file handle based on archive time, and register decompressor if appropriate.
def get_tar_handle ( source )

  type = File.extname( source )

  #switch on last part of filename to determine archive type. Faster than using libmagic, 
  #but obviously less reliable if people do weird things with archives.
  begin
    case type
    when ".tar"
      tar_handle = File.open( source )
    when ".gz"
      gz_file = File.open( source )
      tar_handle = Zlib::GzipReader.new( gz_file )
    when ".bz2"
      bz2_file = File.open ( source )
      tar_handle = StringIO.new ( Bzip2::FFI::Reader.open( bz2_file ).read )
    when ".xz"
      tar_handle = XZ::StreamReader.open( source )
    else
      raise "Unknown archive type: #{ source }"
    end
  rescue
    raise "Error extracting #{ source }"
  end

  #return file handle to decompressor or tar stream
  tar_handle

end

#Extact archive
def extract_archive ( source, dest )

  #get decompressor handle
  tar_handle = get_tar_handle( source )
  #unpack files from archive
  Archive::Tar::Minitar.unpack( tar_handle, dest,
    &lambda do |action, name, stats|
      if action == :file_done or action == :dir
        puts "#{ name }".blue
      end
    end
  )

end

#Generate package tasks for each package.
#Tasks include:
# - A file task for downloading the source package
# - A task that verifies the file against the hash in the project metadata
# - A directory task that extracts the source if it has been downloaded and verified
# - A task that builds & installs the project, using options from each "stage" - i.e. tools build, live root build.
puts "Processing package metadata and building tasks for packages".green
@packages.each do |package|

  #Generate per-package tasks

  #Download package via HTTP
  desc "Download #{ @package_metadata[ package ][ 'file' ] }"
  file "source/#{ @package_metadata[ package][ 'file' ] }" do

    puts "Downloading #{ @package_metadata[ package][ 'file' ] } from #{ @package_metadata[ package][ 'uri' ] }".blue 
    get_package_file( "source/#{ @package_metadata[ package][ 'file' ] }", @package_metadata[ package][ 'uri' ] )

  end

  #Verify source, then extract package by getting appropriate decompressor handle, and passing to minitar.
  #TODO: Write better tar handler to give us more control over decompressor handling, as XZ needs special handling.
  desc "Verify & extract #{ @package_metadata[ package][ 'file' ] } to build/#{ @package_metadata[ package][ 'longname' ] }"
  directory "build/#{ @package_metadata[ package][ 'longname' ] }" => "source/#{ @package_metadata[ package ][ 'file' ] }" do 

    hash = hash_package_file( "source/#{ @package_metadata[ package ][ 'file' ] }" )
    puts "Checking hash for #{ @package_metadata[ package ][ 'file' ] } (#{ hash }) against expected hash (#{ @package_metadata[ package ][ 'hash' ] })".blue
    begin
     raise "Hash failed to verify".red if @package_metadata[ package ][ 'hash' ] != hash 
    end
    puts "Hash verified OK. Going to extract.".blue

    puts "Extracting #{ @package_metadata[ package][ 'file' ] } to build/#{ @package_metadata[ package][ 'longname' ] }".green
    extract_archive( "source/#{ @package_metadata[ package][ 'file' ] }", "build") 
    puts "Extraction finished.".blue
    
  end 

  #Build package based on options in metadata.
  #Accepts package build phase (initial, toolchain, final) as argument.
  if @package_metadata[ package ].key?( 'build' )
    @package_metadata[ package ][ 'build' ].each do |phase,command|
      desc "Build package #{ @package_metadata[ package][ 'name' ] } for phase #{ phase }"
      task "build_#{ phase }_#{ @package_metadata[ package][ 'name' ] }".to_sym do

        #Fetch build command, skip if it doesn't exist for this stage.
        puts "Building #{ @package_metadata[ package ][ 'name' ]} for phase #{ phase }".green
        puts "Running #{ @package_metadata[ package ][ 'build' ][ phase ] } in build/#{ phase }".light_black

        #Make sure build directory exists for this stage
        FileUtils::mkdir_p( "build/#{ phase }/#{ @package_metadata[ package ][ 'longname' ] }" )
 
        #Run build command in appropriate directory
        Dir.chdir( "build/#{ phase }/#{ @package_metadata[ package ][ 'longname' ] }" ){
          system("#{ @package_metadata[ package ][ 'build' ][ phase ] }")
        } 
      end
    end
  else
      puts "No build commands available for package #{ package }"
  end

  desc "Clean package #{ @package_metadata[ package][ 'name' ] }" 
  task "clean_#{ @package_metadata[ package][ 'name' ] }".to_sym do

    #Clean package sources

    #Clean build dir for tools
    
    #Clean build dir for final install

  end

  #Generate temporary dependency on extract task, until these can be codified into stages
  #Eventual dependency chain will be build -> stage task -> build_package -> extract package -> verify source -> download source
  task "#{ package }" => [ "build/#{ @package_metadata[ package][ 'longname' ] }" ] 

end

directory "build/" do

  Dir.mkdir( "build" ) unless File.directory?( "build" )

end

directory "stamps/" do

  Dir.mkdir( "stamps" ) unless File.directory?( "stamps" )

end

directory "output/" do

  Dir.mkdir( "output" ) unless File.directory?( "output" )

end

task :build_root_image do
end

task :build_iso do
end

=begin
task :build_toolchain => [ 
  :build_binutils[ 'initial' ], 
  :build_gcc[ 'initial' ],  
  :build_linux_headers[ 'initial' ],  
  :build_glibc[ 'initial' ], 
  :build_gcc_libstdcpp[ 'initial' ],
  :build_binutils[ 'toolchain' ],
  :build_gcc[ 'toolchain' ],
  :build_ncurses[ 'toolchain' ],
  :build_bash[ 'toolchain' ],
  :build_bzip2[ 'toolchain' ],
  :build_coreutils[ 'toolchain' ],
  :build_diffutils[ 'toolchain' ],
  :build_file[ 'toolchain' ], 
  :build_findutils[ 'toolchain' ],
  :build_gawk[ 'toolchain' ],
  :build_gettext[ 'toolchain' ],
  :build_grep[ 'toolchain' ],
  :build_gzip[ 'toolchain' ],
  :build_m4[ 'toolchain' ],
  :build_make[ 'toolchain' ],
  :build_make[ 'toolchain' ],
  :build_patch[ 'toolchain' ],
  :build_perl[ 'toolchain' ],
  :build_sed[ 'toolchain' ],
  :build_tar[ 'toolchain' ],
  :build_utillinux[ 'toolchain' ],
  :build_xz[ 'toolchain' ]
 ] do
  
  puts "Building toolchain".green  

end 
=end  

task :build_toolchain => [ :build_initial_binutils ] do
end

task :prepare_packages => @packages do

  puts "Checking and preparing package sources".green

end 

task :env do

  ENV['EIR'] = File.dirname(__FILE__)
  puts "Set $EIR environment variable to #{ ENV['EIR'] }".blue

end

task :default => [ :env, "build/", "stamps/", "output/", :prepare_packages, :build_toolchain ]
