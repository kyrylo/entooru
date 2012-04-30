require 'fileutils'
require 'tempfile'

# Path, where all Scriptogram articles are stored.
POSTS_PATH = File.expand_path('~/downloads/.DropboxEntooru/Dropbox/Apps/scriptogram/posts')

namespace :blog do

  desc 'Add all new articles from Github repo to Scriptogram'
  task :add do
    repo_articles    = get_articles('articles')
    dropbox_articles = get_articles(POSTS_PATH)
    new_articles     = repo_articles - dropbox_articles

    unless new_articles.empty?
      new_articles.each do |article|
        puts "\e[1;32mAdding\e[1m \e[0;33m#{ article }\e[0m..."
        convert_for_web(article)
      end

      puts "\n\e[0;32mNew articles have been added!\e[0m"
    end
  end

  desc 'Update a specific article'
  task :update do
    title = ENV['POST']
    unless title
      raise ArgumentError, 'ArgumentError: no POST argument given'
    end

    article = get_articles('articles').find_all { |a| a[title] }

    if article.size != 1
      raise ArgumentError, "ArgumentError: ambiguous argument #{title}"
    end

    article = article[0]

    puts "\e[1;32mUpdating\e[1m \e[0;33m#{ article }\e[0m..."
    convert_for_web(article)
    puts "\n\e[0;32m#{ article } has been added!\e[0m"
  end

  # Retrieve all articles in the given path.
  #
  # path - Place, where your articles are stored.
  #
  # Returns Array of articles in the directory.
  def get_articles(path)
    Dir["#{ path }/*"].map { |article| File.basename(article, '.md') }
  end

  # Convert post to Scriptogram format. Prepends mandatory Scriptogram document
  # header and converts code blocks from Github's flavored Markdown to classic
  # variant.
  #
  # post_name - File to convert.
  #
  # Returns nothing.
  def convert_for_web(post_name)
    post_name.match(/^(\d+)(_.+)$/)
    scriptogram_post = "#{ POSTS_PATH }/#{ post_name }.md"
    post_ru = File.join('articles', post_name, "ru#{ $2 }.md")
    post_en = File.join('articles', post_name, 'en', "en#{ $2 }.md")

    FileUtils.cp(post_ru, scriptogram_post)

    title_ru = File.open(post_ru) { |f| f.readline.chomp }
    title_en = File.open(post_en) { |f| f.readline }

    scriptogram_header =<<-TEXT
---
Date: #{ $1 }
Title: #{ title_ru }
Slug: #{ $1 }-#{ urlify title_en }
---

    TEXT

    File.open(scriptogram_post, 'w') do |sp|
      code = false
      sp.puts scriptogram_header
      File.foreach(post_ru).to_a[3..-1].each do |ru|
        if ru =~ /^```/
          code = code ? false : true
        elsif code
          sp.puts '    ' + ru
        else
          sp.puts ru
        end
      end
    end
  end

  # Convert string to URL format.
  #
  # str - The String to be converted.
  #
  # Examples
  #
  #   watchword = 'En Taro Adun!!!'
  #   url = urlify watchword
  #   # => 'en-taro-adun'
  #
  # Returns converted String, joined by hyphens.
  def urlify(str)
    str.downcase.gsub(/[^a-z0-9]+/i, '-').chomp('-')
  end

end
