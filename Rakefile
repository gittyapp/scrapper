require 'rubygems'

require 'aws-sdk'
require 'jbuilder'
require 'nokogiri'
require 'net/http'
require 'uri'

class Repository
  attr_accessor :owner, :name, :description, :language, :avatar

  def initialize(owner, name, description, language, avatar)
    @owner = owner
    @name = name
    @description = description
    @language = language
    @avatar = avatar
  end

  def to_builder
    Jbuilder.new do |repo|
      repo.owner owner
      repo.name name
      repo.description description
      repo.language language
      repo.avatar avatar
    end
  end
end

def open(url)
  Net::HTTP.get(URI.parse(url))
end

def run_scraper
  languages = ['', 'ABAP','ActionScript','Ada','Apex','AppleScript','Arc','Arduino','ASP','Assembly','Augeas','AutoHotkey','Awk','Boo','Bro','C','csharp','C++','Ceylon','CLIPS','Clojure','COBOL','CoffeeScript','ColdFusion','Common-Lisp','Coq','CSS','D','Dart','DCPU-16-ASM','DOT','Dylan','eC','Ecl','Eiffel','Elixir','Emacs-Lisp','Erlang','F#','Factor','Fancy','Fantom','Forth','FORTRAN','Go','Gosu','Groovy','Haskell','Haxe','Io','Ioke','Java','JavaScript','Julia','Kotlin','Lasso','LiveScript','Logos','Logtalk','Lua','M','Matlab','Max','Mirah','Monkey','MoonScript','Nemerle','Nimrod','Nu','Objective-C','Objective-J','OCaml','Omgrofl','ooc','Opa','OpenEdge-ABL','Parrot','Pascal','Perl','PHP','Pike','PogoScript','PowerShell','Processing','Prolog','Puppet','Pure-Data','Python','R','Racket','Ragel-in-Ruby-Host','Rebol','Rouge','Ruby','Rust','Scala','Scheme','Scilab','Self','Shell','Slash','Smalltalk','Squirrel','Standard-ML','SuperCollider','Tcl','Turing','TXL','TypeScript','Vala','Verilog','VHDL','VimL','Visual-Basic','Volt','wisp','XC','XML','XProc','XQuery','XSLT','Xtend']

  languages.each do |language|
    language.downcase!
    puts "Scraping: #{language}"

    repositories = scrape(language)

    repo_json = Jbuilder.encode do |json|
      json.repositories repositories.each do |repo|
        json.owner repo.owner
        json.name repo.name
        json.description repo.description
        json.language repo.language
        json.avatar repo.avatar
      end
    end

    if language == ''
      language = 'all'
    end

    upload_to_s3(language, repo_json)
  end

  puts 'All Done :D'
end

def write_string_to_file(name, my_string)
  json_file = File.new("json_files/#{name}.json", "w")
  json_file.puts my_string
  json_file.close
end

def upload_to_s3(name, json)
  puts 'Updating json file...'

  access_key = ENV['S3_ACCESS_KEY']
  secret_key = ENV['S3_SECRET_KEY']

  s3 = AWS::S3.new(
  :access_key_id => access_key,
  :secret_access_key => secret_key)

  bucket = s3.buckets['gitty']

  json_file_name = "trending/#{name}.json"
  
  json_file = bucket.objects[json_file_name]

  puts 'Saving Tablet Config...'
  json_file.write(json, :acl => :public_read, :content_type => 'application/json')
end

def scrape(language)
  page_content = open("https://github.com/trending?l=#{language}")

  page = Nokogiri::HTML( page_content )

  repos = page.css('li.repo-leaderboard-list-item')

  repositories = []

  repos.each do |repo|
    main_info = repo.css('a.repository-name')
    repo_owner = main_info.css('span.owner-name').text
    repo_name = main_info.css('strong').text
    repo_description = repo.css('p.repo-leaderboard-description').text
    repo_language = repo.css('span.title-meta').text
    repo_contribs = repo.css('div.repo-leaderboard-contributors a img')
    repo_avatar = ''

    repo_contribs.each do |contrib|
      if contrib['title'] == repo_owner
        repo_avatar = contrib['src']
        repo_avatar = repo_avatar.gsub("s=140", "s=80")
        break
      end
    end

    repository = Repository.new(repo_owner, repo_name, repo_description, repo_language, repo_avatar)

    repositories << repository
  end

  repositories
end

task :run do
  run_scraper
end
