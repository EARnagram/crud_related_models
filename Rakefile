require 'erb'

include FileUtils

def read_contents(file)
  if File.exists? file
    File.read(file).strip
  else
    ""
  end
end

def is_partial(file)
  File.basename(file)[0] == "_"
end

def is_erb(file)
  File.extname(file) == ".erb"
end

def basename_without_erb(file)
  File.basename(file)[0..-5]
end

def erb_parse(file, log=true)
  file_name = file.is_a?(File) ? File.absolute_path(file) : file
  puts "Parsing as ERB: #{file_name}" if log
  ERB.new(read_contents(file_name)).result(binding)
end

def include_partial(file)
  file_name = File.expand_path("../source/#{file}.erb", __FILE__)
  puts "  - Including partial: #{file_name}"
  erb_parse file_name, false
end

desc "Build the files in /source and deposit in /dist"
task :build do
  rm_rf Dir.glob('dist/*')

  mkdir_p "dist/assets"
  cp_r "source/assets", "dist"

  Dir.glob("source/*[^assets]").map { |f| File.new f }.each do |file|
    if is_erb(file) && !is_partial(file)
      contents    = erb_parse file
      destination = "dist/#{basename_without_erb(file)}"
      File.write(destination, contents)
    elsif !is_partial(file)
      cp file, "dist"
      puts "Copied file #{File.absolute_path(file)}"
    end
  end
end
