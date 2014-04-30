require 'rubygems'

require 'aws-sdk'
require 'jbuilder'
require 'nokogiri'
require 'net/http'
require 'octokit'
require 'uri'

class Repository
  attr_accessor :owner, :name, :description, :language, :avatar, :stars, :forks

  def initialize(owner, name, description, language, avatar, stars, forks)
    @owner = owner
    @name = name
    @description = description
    @language = language
    @avatar = avatar
    @stars = stars
    @forks = forks
  end

  def to_builder
    Jbuilder.new do |repo|
      repo.owner owner
      repo.name name
      repo.description description
      repo.language language
      repo.avatar avatar
      repo.stars stars
      repo.forks forks
    end
  end
end

class User
  attr_accessor :login, :name, :avatar, :repo

  def initialize(login, name, avatar, repo)
    @login = login
    @name = name
    @avatar = avatar
    @repo = repo
  end  
end

def open(url)
  Net::HTTP.get(URI.parse(url))
end

def run_scraper
  languages = ['', 'ABAP','ActionScript','Ada','Apex','AppleScript','Arc','Arduino','ASP','Assembly','Augeas','AutoHotkey','Awk','Boo','Bro','C','csharp','C++','Ceylon','CLIPS','Clojure','COBOL','CoffeeScript','ColdFusion','Common-Lisp','Coq','CSS','D','Dart','DCPU-16-ASM','DOT','Dylan','eC','Ecl','Eiffel','Elixir','Emacs-Lisp','Erlang','F#','Factor','Fancy','Fantom','Forth','FORTRAN','Go','Gosu','Groovy','Haskell','Haxe','Io','Ioke','Java','JavaScript','Julia','Kotlin','Lasso','LiveScript','Logos','Logtalk','Lua','M','Matlab','Max','Mirah','Monkey','MoonScript','Nemerle','Nimrod','Nu','Objective-C','Objective-J','OCaml','Omgrofl','ooc','Opa','OpenEdge-ABL','Parrot','Pascal','Perl','PHP','Pike','PogoScript','PowerShell','Processing','Prolog','Puppet','Pure-Data','Python','R','Racket','Ragel-in-Ruby-Host','Rebol','Rouge','Ruby','Rust','Scala','Scheme','Scilab','Self','Shell','Slash','Smalltalk','Squirrel','Standard-ML','SuperCollider','Tcl','Turing','TXL','TypeScript','Vala','Verilog','VHDL','VimL','Visual-Basic','Volt','wisp','XC','XML','XProc','XQuery','XSLT','Xtend']

  access_token = ENV['GH_ACCESS_TOKEN']

  client = Octokit::Client.new(:access_token => access_token)

  languages.each do |language|
    language.downcase!
    puts "Scraping: #{language}"

    repositories = scrape_repos(language, client)

    repo_json = Jbuilder.encode do |json|
      json.repositories repositories.each do |repo|
        json.owner repo.owner
        json.name repo.name
        json.description repo.description
        json.language repo.language
        json.avatar repo.avatar
        json.watchers repo.stars
        json.forks repo.forks
      end
    end

    users = scrape_users(language)

    user_json = Jbuilder.encode do |json|
      json.users users.each do |user|
        json.login user.login
        json.name user.name
        json.avatar user.avatar

        json.repository do
          json.owner user.repo.owner
          json.name user.repo.name
          json.description user.repo.description
        end
      end
    end

    if language == ''
      language = 'all'
    end

    upload_to_s3(language, repo_json, 'repo')
    upload_to_s3(language, user_json, 'user')
  end

  puts 'All Done :D'
end

def write_string_to_file(name, my_string)
  json_file = File.new("json_files/#{name}.json", "w")
  json_file.puts my_string
  json_file.close
end

def upload_to_s3(name, json, prefix)
  puts 'Updating json file...'

  access_key = ENV['S3_ACCESS_KEY']
  secret_key = ENV['S3_SECRET_KEY']

  s3 = AWS::S3.new(
  :access_key_id => access_key,
  :secret_access_key => secret_key)

  bucket = s3.buckets['gitty']

  json_file_name = "trending/#{prefix}/#{name}.json"
  
  json_file = bucket.objects[json_file_name]

  puts "Saving JSON File #{json_file_name}..."
  json_file.write(json, :acl => :public_read, :content_type => 'application/json')
end

def scrape_repos(language, client)
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

    gh_user = client.user repo_owner
    repo_avatar = gh_user.avatar_url

    stars_css = repo.css('span.collection-stat')[0]
    repo_stars = stars_css.text unless stars_css == nil

    forks_css = repo.css('span.collection-stat')[1]
    repo_forks = forks_css.text unless forks_css == nil

    repository = Repository.new(repo_owner, repo_name, repo_description, repo_language, repo_avatar, repo_stars, repo_forks)

    repositories << repository
  end

  repositories
end

def scrape_users(language)
  page_content = open("https://github.com/trending/developers?l=#{language}")

  page = Nokogiri::HTML( page_content )

  page_users = page.css('li.user-leaderboard-list-item')

  users = []

  page_users.each do |user|
    main_info = user.css('div.leaderboard-list-content')
    user_login = main_info.css('h2.user-leaderboard-list-name a')[0]['href'].gsub("/", "")
    user_name = main_info.css('span.full-name').text.gsub("(", "").gsub(")", "")
    user_avatar = user.css('img.leaderboard-gravatar')[0]['src']
    repo_description = main_info.css('span.repo-description').text
    repo_name = main_info.css('span.repo').text
    
    repo = Repository.new(user_login, repo_name, repo_description, '', user_avatar, '', '')
    user = User.new(user_login, user_name, user_avatar, repo)

    users << user
  end

  users
end

task :run do
  run_scraper
end
