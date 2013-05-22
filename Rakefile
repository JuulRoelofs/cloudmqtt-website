require 'haml'
require 'redcarpet'
require 'aws'
require 'mime/types'
require 'fileutils'

task :start do
  exec 'ruby app.rb'
end

desc 'Render all haml files and copy public files to output'
task :render => :clean do
  FileUtils.cp_r 'public/.', 'output'

  haml_options = { :format => :html5 }
  haml_layout = File.read('views/layout.haml')
  layout = Haml::Engine.new(haml_layout, haml_options)
  Dir['views/*.haml'].each do |f|
    name = File.basename(f, '.haml')
    next if name == 'layout' or name.start_with? '_'

    haml_view = File.read(f)
    view = Haml::Engine.new(haml_view, haml_options)
    ctx = ViewCtx.new(haml_options)
    inner = view.to_html(ctx, {name: name})
    html = layout.to_html(ctx, {name: name}) do
      inner
    end
    File.open("output/#{name}.html", 'w+') {|o| o.write html}
  end
end

class ViewCtx
  def initialize(opts)
    @opts = opts

    rnder = Redcarpet::Render::HTML.new(prettify: true)
    @markdown = Redcarpet::Markdown.new(rnder, {
      :autolink => true,
      :space_after_headers => true,
      :no_intra_emphasis => true,
      :fenced_code_blocks => true,
      :space_after_headers => true
    })
  end

  def haml(view_sym, opts)
    haml_view = File.read("views/#{view_sym}.haml")
    engine = Haml::Engine.new(haml_view, @opts.merge(opts))
    engine.to_html(ViewCtx.new(@opts), opts[:locals])
  end

  def markdown(view_sym)
    view = File.read("views/#{view_sym}.md")
    @markdown.render(view)
  end
end

desc 'Recreate the output folder'
task :clean do
  FileUtils.rm_rf 'output'
  FileUtils.mkdir_p 'output'
end

task :gzip => :render do
  files = Dir['output/**/*'].select{ |f| File.file? f }
  files.each do |f|
    ct = MIME::Types.of(f).first.to_s
    next unless ct =~ /^text|javascript$/
    Zlib::GzipWriter.open("#{f}.gz") do |gz|
      gz.write IO.binread(f)
    end
    size = File.size f
    gzip_size = File.size "#{f}.gz"
    puts "Compressing: #{f} saving #{(size - gzip_size)/1024} KB"
    FileUtils.rm f
    FileUtils.mv "#{f}.gz", f
  end
end

desc 'Sync output with S3 bucket'
task :upload => :gzip do
  s3 = AWS::S3.new(YAML.load(File.read('aws.yml')))
  objects = s3.buckets['www.cloudmqtt.com'].objects
  files = Dir['output/**/*'].select{ |f| File.file? f }

  objects.each do |obj|
    if f = files.find {|fn| fn == "output/#{obj.key}" }
      md5 = Digest::MD5.file(f).to_s
      if not obj.etag[1..-2] == md5
        ct = MIME::Types.of(f).first.to_s
        ct = "text/html;charset=utf-8" if ct == "text/html"
        ce = 'gzip' if ct =~ /^text|javascript$/
        puts "Updating: #{f} Content-type: #{ct} Content-encoding: #{ce}"
        objects[f.sub(/output\//,'')].write(:file => f, :content_type => ct, content_encoding: ce)
      else
        puts "Not changed: #{f}"
      end
      files.delete f
    else
      obj.delete
      puts "Deleting: #{obj.key}"
    end
  end

  files.each do |f|
    ct = MIME::Types.of(f).first.to_s
    ct += ";charset=utf-8" if ct == "text/html"
    puts "Uploading: #{f} Content-type: #{ct} Content-encoding: #{ce}"

    ce = 'gzip' if ct =~ /^text|javascript$/
    objects[f.sub(/output\//,'')].write(:file => f, :content_type => ct, content_encoding: ce)
  end
end
