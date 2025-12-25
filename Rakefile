# frozen_string_literal: true

namespace :vendor do
  task :data do
    module Tainted
      module_function

      MANUAL_TAINTED = {
        "The Lost" => "Tainted Lost",
        "The Forgotten" => "Tainted Forgotten",
        "Jacob and Esau" => "Tainted Jacob",
      }

      def fetch(name)
        MANUAL_TAINTED.fetch(name) { "Tainted #{name}" }
      end
    end

    class RequirementsScrubber
      class << self
        def call(text)
          if text.include?("Removed in") && text.include?("Added in")
            i_start = text.index("Removed in")
            i_end = text.rindex("Added in") - 1

            text.slice!(i_start..i_end)
          end

          text.gsub!(/\[\[[fF]ile.*?\]\]/, "")

          text.gsub!(/\[\[(.*?|)?([^|]*?)\]\]/, '[[\2]]')

          text.gsub!(/<\/?span.*?>/, "")

          text.squeeze!(" ")
          text.lstrip!

          text
        end
      end
    end

    class PrereqParser
      NONE = [].freeze

      CHALLENGE_REGEX = /Complete \[\[(.*?) \(challenge #/
      DEFEAT_AS_REGEX = /^Defeat \[\[(.*?)\]\] as \[\[(.*?)\]\]/
      OTHER_AS_REGEX = /\[\[(Home|Boss Rush|Completion Mark)\]\]s? as \[\[(.*?)\]\]/

      MANUAL_PREREQ_MAP = {
        "Isaac" => nil,
      }

      MANUAL_IDS = {
        "82" => ["Missing Poster"],
        # Default Challenges
        "60" => NONE,
        "63" => NONE,
        "89" => NONE,
        "90" => NONE,
        "91" => NONE,
        "100" => NONE,
        "101" => NONE,
        "102" => NONE,
        "103" => NONE,
        "104" => NONE,
        "517" => NONE,
        # Mis-cased challenges
        "332" => ["Aprils fool"],
        "335" => ["PONG"],
        "532" => ["Cantripped!"],
        # Mom's Heart #s
        "8" => ["Rubber Cement"],
        "159" => ["Rubber Cement"],
        "139" => ["A Noose"],
        "33" => ["Wire Coat Hanger"],
        "140" => ["Everything Is Terrible!!!"],
        "141" => ["Ipecac"],
        "10" => ["Experimental Treatment"],
        "11" => ["A Quarter"],
        "162" => ["A Quarter"],
        "32" => ["A Fetus in a Jar"],
        "234" => ["A Fetus in a Jar"],
        # It Lives! #s
        "343" => ["???"],
        "344" => ["Flooded Caves"],
        "345" => ["Dank Depths"],
        # Hush
        "320" => ["Blue Womb"],
        "407" => ["New Area"],
      }

      def initialize(names)
        @names = names
        @prereq_map = MANUAL_PREREQ_MAP.dup
      end

      def call(achievement)
        id = achievement["id"]

        MANUAL_IDS.fetch(id) { match(achievement) }.tap do |parsed_names|
          puts "Unknown name for id #{id}: #{parsed_names}" unless parsed_names.all? { |n| @names.include? n }
        end
      end

      private

      def match(achievement)
        text = achievement["requirements"]

        prereqs = case text
        when CHALLENGE_REGEX
          [$1]
        when DEFEAT_AS_REGEX
          case $1
          when "Hush"
            ["Blue Womb", $2]
          else
            [$2]
          end
        when OTHER_AS_REGEX
          case $1
          when "Home"
            @prereq_map[Tainted.fetch($2)] = achievement["name"]
          end

          [$2]
        when /Chapter 4/
          ["The Womb"]
        when /11 times/
          ["???"]
        else
          []
        end

        prereqs.filter_map { |p| @prereq_map.fetch(p) { p } }
      end
    end

    require "net/http"
    require "json"
    require "uri"

    achievements = []

    base_uri = "https://bindingofisaacrebirth.wiki.gg/api.php?action=cargoquery&tables=achievement&format=json&fields=id,name,requirements&limit=500"

    [URI.parse(base_uri), URI.parse("#{base_uri}&offset=500")].each do |uri|
      response = Net::HTTP.get_response(uri)

      case response
      when Net::HTTPSuccess
        cargoquery = JSON.parse(response.body)

        $stdout.write "."

        achievements.concat(cargoquery["cargoquery"].map { |q| q["title"] })
      else
        $stdout.write "F"
      end
    end

    $stdout.puts

    prereq_parser = PrereqParser.new(achievements.map { |a| a["name"] })

    achievements.each do |a|
      a["requirements"] = RequirementsScrubber.call(a["requirements"])
      a["prereqs"] = prereq_parser.call(a)

      a["requirements"].gsub!(/\[\[|\]\]/, "")
    end

    JSON.dump(achievements, File.open("achievements.json.tmp", "w"))

    sh "jq . achievements.json.tmp > achievements.json"

    rm "achievements.json.tmp"
  end
end
