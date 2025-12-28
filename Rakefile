# frozen_string_literal: true

DONATION_MACHINE = [
  "Blue Map",            # 10
  "Store Upgrade lv.1",  # 20
  "There's Options",     # 50
  "Store Upgrade lv.2",  # 100
  "Black Candle",        # 150
  "Store Upgrade lv.3",  # 200
  "Red Candle",          # 400
  "Store Upgrade lv.4",  # 600
  "Blue Candle",         # 900
  "Stop Watch",          # 999
]

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
      DEFEAT_AS_REGEX = /^Defeat \[\[(.*?)\]\](?:.* as \[\[(.*?)\]\])?/
      OTHER_AS_REGEX = /\[\[(Home|Boss Rush|Completion Mark)\]\]s? as \[\[(.*?)\]\]/

      MANUAL_PREREQ_MAP = {
        "Isaac" => nil,
      }

      MANUAL_NAMES = {
        "The Lost" => ["Missing Poster"],
        # Default Challenges
        "Burnt Penny" => NONE,
        "SMB Super Fan" => NONE,
        "Rune of Hagalaz" => NONE,
        "Rune of Jera" => NONE,
        "Rune of Ehwaz" => NONE,
        "Card Against Humanity" => NONE,
        "Swallowed Penny" => NONE,
        "Robo-Baby 2.0" => NONE,
        "Death's Touch" => NONE,
        "Technology .5" => NONE,
        "Dirty Mind" => NONE,
        # Mis-cased challenges
        "Maggy Now Holds a Pill!" => ["Aprils fool"],
        "Greed's Gullet" => ["PONG"],
        "Justice" => ["Cantripped!"],
        # Mom's Heart #s
        "A Noose" => ["Rubber Cement"], # 3
        "Solar System" => ["Rubber Cement"],
        "Wire Coat Hanger" => ["A Noose"], # 4
        "Everything Is Terrible!!!" => ["Wire Coat Hanger"], # 5
        "Ipecac" => ["Everything Is Terrible!!!"], # 6
        "Experimental Treatment" => ["Ipecac"], # 7
        "A Quarter" => ["Experimental Treatment"], # 8
        "A Fetus in a Jar" => ["A Quarter"], # 9
        "Demo Man" => ["A Quarter"],
        "???" => ["A Fetus in a Jar"], # 10
        "Blue Womb" => ["A Fetus in a Jar"],
        # It Lives! #s
        "Flooded Caves" => ["???"],
        "Dank Depths" => ["Flooded Caves"],
        "Scarred Womb" => ["Dank Depths"],
        # Hush
        "A Secret Exit" => ["New Area"],
        # Corpse/Mother
        "Rotten Heart" => ["A Secret Exit"],
        "Dross" => ["A Secret Exit"],
        "Ashpit" => ["A Secret Exit"],
        "Gehenna" => ["A Secret Exit"],
        # Home
        "Red Key" => ["A Strange Door"],
        # Complex/Multi Requirement
        "Darkness Falls" => ["???", "Eve"],
        "Baptism by Fire" => ["Bethany", "???", "Magdalene"],
      }
      DONATION_MACHINE.each_with_index do |name, i|
        next unless prev = DONATION_MACHINE[i - 1]
        MANUAL_NAMES[name] = [prev]
      end

      BOSSES = {
        "Delirium" => "New Area",
        "Hush" => "Blue Womb",
        "Mega Satan" => "Angels",
        "Mother" => "A Secret Exit",
        "The Beast" => "A Strange Door",
        "Ultra Greedier" => "Greedier!",
      }

      def initialize(names)
        @names = names
        @prereq_map = MANUAL_PREREQ_MAP.dup
      end

      def call(achievement)
        name = achievement["name"]

        MANUAL_NAMES.fetch(name) { match(achievement) }.tap do |parsed_names|
          puts "Unknown prereq for #{name}: #{parsed_names}" unless parsed_names.all? { |n| @names.include? n }
        end
      end

      private

      def match(achievement)
        text = achievement["requirements"]

        prereqs = case text
        when /11 times/ # Before DEFEAT_AS to catch "Defeat Mom's Heart 11 times"
          ["???"]
        when CHALLENGE_REGEX
          [$1]
        when DEFEAT_AS_REGEX
          [BOSSES[$1], $2].compact
        when OTHER_AS_REGEX
          case $1
          when "Home"
            @prereq_map[Tainted.fetch($2)] = achievement["name"]

            [$2, "A Strange Door"]
          else
            [$2]
          end
        when /Chapter 4/
          ["The Womb"]
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

  namespace :prio do
    task :shrubs do
      # https://steamcommunity.com/sharedfiles/filedetails/?id=3434706205

      require "json"

      priority = {}
      output = {
        rank: [
          :misc,
          :daily,
          :runes,
          :challenges,
          "early progression",
          :characters,
          :unknown,
          :bad,
        ],
        priority:,
      }

      [
        *DONATION_MACHINE,
        "The Lost",
        "Hat trick!", # 3 win streak
        "5 Nights at Mom's", # 5 win streak with different characters
        "A Halo",
        "Once More With Feeling!", # Victory Lap
        "Marbles",
        "Huge Growth",
        "Counterfeit Coin",
        "The Planetarium",
        "Technology Zero",
      ].each { priority[it] = "misc" }

      [
        "The Marathon",
        "Broken Modem",
        "Dedication", # Technically bad but takes 31 days
      ].each { priority[it] = "daily" }

      [
        "Rune of Jera",
        "Rune of Ehwaz",
        "Rune of Ansuz",
        "Blank Rune",
        "Rune of Perthro",
        "Rune of Dagaz",
        "Rune of Algiz",
      ].each { priority[it] = "runes" }

      [
        "Card Against Humanity",
        "Burnt Penny",
        "SMB Super Fan",
        "Death's Touch",
        "Technology .5",
        "Chaos Card",
        "Credit Card",
        "Epic Fetus",
        "Gold Heart",
        "Get out of Jail Free Card",
        "Gold Bomb",
        "Laz Bleeds More!",
        "Maggy Now Holds a Pill!",
        "Charged Key",
        "Samson Feels Healthy!",
        "Sigil of Baphomet",
        "Spirit Sword",
        "Justice",
      ].each { priority[it] = "challenges" }

      [
        "Lucky Pennies",
        # Store Upgrade v1 (-> misc)
        "The Polaroid",
        "The Negative",
        "New Area",
        "Missing Poster",
        "A Secret Exit",
        "A Strange Door",
        "Greedier!",
      ].each { priority[it] = "early progression" }

      [
        "Magdalene",
        "Cain",
        "Eve",
        "Samson",
        "Azazel",
        "Lazarus",
        "Judas",
        "Eden",
        "Bethany",
        "???",
        # The Lost (-> misc)
        "Lilith",
        "The Keeper",
        "Apollyon",
        "The Forgotten",
        "Jacob and Esau",
      ].each { priority[it] = "characters" }

      [
        "Corrupted Data",
        "2 new pills",
        "Rules Card",
        "Suicide King",
        "Mr. Resetter!",
        "The Scissors",
        "Abel",
        "Missing No.",
        "???'s Only Friend",
        "Isaac's Head",
        "Shade",
        "Rune of Berkano",
        "TMTRAINER",
        "Torn Card",
        "IBS",
        "The High Priestess",
      ].each { priority[it] = "bad" }

      JSON.dump(output, File.open("priority-shrubs.json.tmp", "w"))

      sh "jq . priority-shrubs.json.tmp > priority-shrubs.json"

      rm "priority-shrubs.json.tmp"
    end
  end
end
