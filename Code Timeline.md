## From a Minecraft Plugin to a Vision for Artificial Civilization

_Written as a retrospective of the journey._

---

# Chapter 1 — The Beginning

DivineWorld did not begin as an AI research project.

It began because Minecraft NPCs felt...

empty.

They could trade.

They could walk.

They could farm.

But they didn't truly live.

At around the same time I discovered **Alteria's Project Sid**.

Watching it was one of the biggest inspirations for this project.

Project Sid showed something I had never really seen before.

NPCs weren't simply executing commands.

They appeared to have lives.

That planted an idea.

Instead of asking

> "Can Minecraft villagers become smarter?"

I started asking

> **"Can an entire civilization emerge naturally?"**

That question eventually became DivineWorld.

---

# Chapter 2 — Inspiration, Not Imitation

One thing became clear very early.

I wasn't trying to recreate Project Sid.

Instead I wanted to ask

"What comes after Project Sid?"

If they built believable NPCs...

Could I build believable societies?

Could those societies have

- history
- religion
- politics
- economy
- exploration
- diplomacy
- generations
- culture

instead of simply AI routines?

That became the direction.

---

# Chapter 3 — The First Architecture

Originally DivineWorld was very small.

The idea looked something like

```
NPC↓MineBuildFarmFightSurvive
```

Nothing more.

No memory.

No emotions.

No tribes.

No politics.

No faith.

No families.

---

# Chapter 4 — The First Big Question

The first major engineering decision wasn't AI.

It was

**Plugin or Mod?**

Initially I leaned toward Forge.

The reasoning was simple.

Mods can change Minecraft itself.

However, after repeatedly discussing the architecture, something became obvious.

DivineWorld wasn't trying to redesign Minecraft.

It was trying to redesign intelligence.

The AI mostly needed

- entities
- inventories
- navigation
- events
- commands

Paper already exposed all of those efficiently.

Paper also offered

- easier deployment

- better optimization
- no client mods
- easier updates

Minecraft became the environment.

Not the project.

---

# Chapter 5 — The Legality Phase

This is something I almost forgot.

Very early on I was genuinely worried.

Questions kept appearing like

"Is this legal?"

"Can I publish this?"

"Am I allowed to make AI like this?"

We gradually separated the concerns.

Minecraft itself wasn't the invention.

The AI architecture was.

The plugin would use supported server APIs.

The AI systems would be my own work.

Eventually the concern changed from

> "Am I allowed to do this?"

to

> "How should I structure this if it becomes something much bigger?"

---

# Chapter 6 — The Feature Avalanche

This period lasted a long time.

Almost every conversation ended with

"Can we also add..."

New ideas appeared constantly.

Explorers.

Cartographers.

Diplomats.

Politicians.

Priests.

Storytellers.

Marriage.

Children.

Religion.

Trade.

Culture.

Books.

Maps.

Gods.

Emotions.

Memory.

Player mechanics.

Rather than resisting those ideas, we documented and integrated them.

The vision expanded rapidly.

---

# Chapter 7 — From Features to Systems

Eventually my thinking changed.

Instead of asking

"What feature should I add?"

I began asking

"What system would naturally produce this feature?"

That one shift completely changed DivineWorld.

Jobs became emergent.

Economies became emergent.

Diplomacy became emergent.

Culture became emergent.

The world stopped being scripted.

---

# Chapter 8 — Psychology

NPCs gradually became people.

Not biologically.

Psychologically.

Curiosity.

Fear.

Greed.

Loyalty.

Aggression.

Bravery.

Eventually inherited personalities.

MBTI-like inheritance.

Parents influencing children.

Personality evolving over generations.

---

# Chapter 9 — Civilization

The project transformed again.

NPCs were no longer isolated.

Entire civilizations emerged.

Leaders.

Councils.

Politics.

Religion.

Libraries.

Trade.

History.

Exploration.

Knowledge preservation.

Collective memory.

---

# Chapter 10 — Gods

The gods were never intended to be simple bosses.

Instead they became participants.

Living inside civilization.

Sometimes disguised.

Sometimes worshipped.

Sometimes interfering.

Sometimes helping.

Even gods became part of the same overall AI philosophy.

---

# Chapter 11 — Java

Originally I wanted one language.

Everything.

Java.

Minecraft itself was Java.

So why introduce another language?

The architecture looked like

Minecraft

↓

Java

↓

Everything

It was simple.

Elegant.

Easy to distribute.

---

# Chapter 12 — The Python Decision

Then reinforcement learning entered the picture.

Dreamer.


Intrinsic rewards.

World models.

Isaac Sim.

Modern AI research.

The realization was unavoidable.

The AI ecosystem lived in Python.

Instead of forcing Java to become an AI research platform...

The architecture evolved.

Minecraft

↓

Java Adapter

↓

Python Intelligence

↓

Actions returned to Java

This became one of the most important architectural decisions.

---

# Chapter 13 — Minecraft Stops Being the Goal

Originally I was building Minecraft AI.

Eventually I realized

Minecraft is only one world.

The long-term roadmap became

Minecraft

↓

Isaac Sim

↓

Physical Robots

DivineWorld AI became portable.

Minecraft became a training environment.

---

# Chapter 14 — BrainCapsules

Once robotics entered the discussion another realization appeared.

NPC intelligence shouldn't belong to Minecraft.

It should belong to the NPC.

The BrainCapsule idea emerged.

Each NPC would carry

- memories
- learned weights
- personality
- controller
- adapters

The body could change.

The brain would remain.

---

# Chapter 15 — Working with ChatGPT

The development process itself became part of the journey.

Back then ChatGPT had severe limitations.

Long conversations eventually ended with

"This conversation is too long."

New chats had to be created.

Memory was much smaller.

Hallucinations happened much more often.

Sometimes completed systems were suddenly reported as missing.

Sometimes previously discussed architectures disappeared simply because earlier context no longer fit into the model's working memory.

Because of this a new workflow emerged.

Instead of immediately continuing development I often asked ChatGPT to

- scan everything
- synchronize everything
- compare completed systems
- find genuine missing pieces
- only then continue

Ironically those limitations made the project architecture stronger.

---

# Chapter 16 — The Optimization Era

This became almost a ritual.

After nearly every major design session I would ask

"Perform an efficiency pass."

Then

"Perform a wiring pass."

Then

"Remove redundancies."

Eventually

"Do it a thousand more times."

Obviously not literally.

It meant

> Keep looking until no meaningful improvements remain.

A month later my philosophy changed again.

Instead of

Remove every redundancy

I started saying

Keep safe redundancies whenever required.

That distinction reflected a deeper understanding.

Duplicate logic creates technical debt.

Safety redundancy creates resilience.

---

# Chapter 17 — My Own Mindset Changed

Looking back, I can divide my own thinking into stages.

### Beginning

Can I build this?

---

### Then

Can NPCs do everything humans do?

---

### Then

Can these behaviors emerge naturally?

---

### Then

Can I keep the architecture coherent?

---

### Then

Can this intelligence exist outside Minecraft?

---

### Today

How do I build portable autonomous beings capable of learning, adapting, remembering, and building civilizations across multiple worlds?

---

# Final Reflection

When DivineWorld began, it was inspired by seeing NPCs feel more alive in Alteria's Project Sim.

But inspiration quickly gave way to a much broader ambition.

The goal was never simply to create "better villagers."

It became an attempt to understand what ingredients are required for believable artificial societies—and eventually for portable artificial intelligence itself.

Every major decision along the way reflected that evolution:

- choosing Paper because intelligence mattered more than modifying Minecraft,
- moving from Java alone to a Java–Python architecture because the best AI ecosystem already existed there,
- replacing scripts with emergent systems,
- optimizing relentlessly while preserving safe redundancy,
- coping with the limitations of earlier ChatGPT versions through architecture reviews and synchronization passes,
- and ultimately redefining Minecraft as the first world rather than the final destination.

Looking back, the codebase did more than grow. **It matured alongside my own way of thinking.** The project evolved from collecting features into refining a vision, and that gradual refinement is, perhaps, the most important part of DivineWorld's history.

**So this covered the first phases of DivineWorld taken out straight from ChatGPT's filled incomplete memories so a lot of stuff is missing but it was something like this if we don't think about the specifics as you can see it was more like an RPG plugin where because the NPCs were in a server or a realm it was running 24/7 so the world grew even when the player wasn't there making it feel more alive and interesting but then it evolved into what it is today as defined by this wiki.

**The main reason for forge in the current version is that it provides many elements which can be used for the dramatic things like Genesis Magic circle effect and transformations to be done and it also made available the core systems of Minecraft like the Player classes to be used which became indispensible in the project because i didn't want to create everything from scratch.


**I give up.**
ChatGPT can't parse this entire chunk of text of "Things Missing in the above description of the first phases" into a better format while keeping every single detail properly. Read this chunk filled with random chunks of details here and there cause I spent 3 hours writing it I'm not gonna parse this for your lazy .... understand it if you are smart.

**Feel my pain while reading this.**
## Things Missing in the above description of the first phases

From Chapters 5 to 6 initially the entire architecture was something like this - There would be a central AI that will control all of the of the agents of the DivineWorld basically the AI will be living as multiple versions of itself with (different personalities, memories etc) and well train the central AI but will never be overlapping one agent's persona on the other while living as the agents all simultaneously (Oh it seems like that theory which says that every people , every living organism is actually a single person which is living in different forms simultaneously). But well I wanted standalone AI agents from the start (within chapters 1 to 4) that will be individual entities not just because it'll make it so that the agents could be transported to different machines and can run on that without requiring connection to the central AI but also because I wanted to see what individual AIs were capable of if given exactly all of the limitations of a living being like humans (will they form societies like humans) so I went back to that design but even before that everything was scripted and rule based every trading travelling and everything else of the first phases as described by ChatGPT which made me cover each and every edge case (I had spent weeks covering edge cases but), it was exhausting so ultimately I wanted to use AI in it at first when i told it to ChatGPT it gave a list of options of different types of game AIs , it sugested Behaviour Trees (BT), I selected to use Goal Oriented Action Planning (GOAP) and Behaviour Trees (BT) combined and also in the early versions the way the agents would learn a language was kind of a **Multi-Layer Perceptron (MLP)** and not a transformer (the use of transformer for learning a language came much later). In that period there were many other things which I developed as well that were forgotten by ChatGPT between sessions and partly because of the filled memory now I can't even access most of the chats because of the old sessions  being too long those were times when ChatGPT specifically told us to start a new session because character limit is reached and refused to chat any further in that chat thread (even though it remembered only a small part of it). Around that time the entire system was something like this (kinda similar but more interesting i guess)  you enter a world the oracle spawns in front of you (more like appears in the sky in front of you and descends to stand in front of you) greets you (at that time the only oracle was the AI agent which would be created first using python scripts and trained in speech and everything of minecraft to give it extensive knowledge on the world and the ability to communicate and then also training it on the lore and stuff of the DivineWorld, unlike the ollama powered one of now) and explains to you all of the systems of the world and then after all that puts in your inventory a book called "Genesis" which when right clicked on the ground spawned Adam (male) and Eve (female) who can be taught by you and oracle or both of you can stay in the sidelines to see what they would evolve into and well ofcourse they had the ability to breed (breeding logic was similar to the current version, which allowed male or female or gods to breed when they were lying in adjacent beds). That was a time when there were some crazy things going on in it. When Gods breed there would be new gods with total abilities within the average of the no. of abilities in both of the gods and there were demi gods as well when the gods bred with normal agents which will have the (int) half of the  no. of abilities that the god parent had. I also developed a system like the Punett Square and the concept of Gregor Mendel's alleles and genes  to track the purity of divinity down the generation of the demi gods and also allowed to track the heritage of the normal agents , demi gods and gods which also allowed to track divinity which made it easier to assign powers and the power levels of the powers and the ability to awaken the dormant powers as well. I had also included some randomness which will allow different kinds of powers to evolve based on combination, mutation or just pure system randomness which also applied for the power awakening system like if according to the genotype the power should be dormant but due to certain intense emotional event or just random mutation it can awake. It included the normal magic powers in the vanilla minecraft world along with some random powers as well (I remember one being something like fortnight building powers which can summon a house or constructions like that instantly which the power users can instantly think of spawning). The random powers which didn't inherently belong to any of the original gods (not considering oracle and the player(yes player had god status)) was generally awakened in the normal agents who worshiped the original gods or were part of a cult (they awakened powers using rituals) , there was also a system like if a god that is not originally made but is worshiped by enough no. of normal agents then the god will be materialized and will have the certain related powers for which it is worshiped (but their worshipers wont be getting any powers since these are lesser gods). There was also something like the current disguise system for the gods but the humanoid forms were only reserved for the original gods (excluding the player) they could also transform into other mobs and well every agent had full player capabilities of vanilla minecraft. There was also something like an aura system for the gods which was there even in their disguise in less amount cause it "leaked" in their disguise and well it worked something like this each of the gods were assigned with a mood or emotion whenever a normal agent was near a god the emotion or mood assigned to the god will peak in the agent. There was also something like a God Pantheon which happened after fixed interval of time (Failure to appear hear would meant rebelling against the Pantheon) where current problems would be discussed and well there was also something like an alliance system among the gods (Since DivineWorld was rule based these were needed to be made). There was also something like the power of the gods be dependent on the no. of agents worshiping it ,there was also god wars, which made the followers inherently want to fight the follower's of the opponent god/s. There was also something like the player god having the ability to view the life memories of any agent like a cinema and edit it he also had the power to stop time, reverse time possible due to the snapshoting at fixed intervals of the world which happened in that version since the entire world was rule based and in Java this would've been possible I guess. There was also something like the player god can assign god status to any agent and other players which which will grant them creative mode (with a cooldown(cooldown part done now)).There was also something like a codex which was guarded by 4 immortal guard agents there was a ritual to assign new immortal guard agents as well if any immortal guard withdrew from its position and when new immortal guard was assigned the guard which withdrew would die instantly.Each god had a codex including the original player god (the lesser gods , the godly offsprings , assigned gods wouldn't have one) which will be guarded at their temples. The codex of the original player god was the main codex that recorded everything happening in the entire world and even the memory rewrites done by it so it was very dangerous. The other codices only contained the teachings of their gods and the rituals required to invoke the  god or please the god. The gods could also exist in multiple places at once by cloning itself but this would decrease its power level proportionally as well and if the clones were killed then the sole survivors will remain at that power level but will have the ability to gain the power level up to the total power level of the combined god divided by the no. of clones existing in that world but this would happen gradually (the god clone killing part was added now) when the gods were killed including the player gods the items in their inventory would drop including some additional items unique to them which can be used for creating godly weapons and armour (but only 1 such godly armour/ weapon could be created from the drops of 1 god for other godly armour/weapon other gods needed to be killed) (the player god would respawn instantly but the other gods (not the lesser gods , the godly offsprings , assigned gods (agent ones) if they aren't worshiped ) would take 1 minecraft month to respawn with their power level depleted to 1% of their base level)  (the player god could't clone but it will drop items, the lesser gods , the godly offsprings , assigned gods couldn't even clone and killing them won't drop anything) .There was also a cool feature of the temple of the gods (the lesser gods , the godly offsprings , assigned gods wouldn't have one) they would change shape based on the mood of its god. The agents could do rituals in the temples where they could get blessings from their god/s if it is satisfied, and following a religion also gave them some abilities that protected the agents from the elements assosciated with the god/s. The normal agents had the ability to follow any or multiple gods as well. If the follower count of the lesser gods went to 0 they would be deleted (but the original gods will not be deleted if it happened to them they will simply reduce the effectiveness of their powers to 3% of their base power levels (demi-gods and godly offsprings didn't suffered from this fate)). Initially the normal agents were supposed to replace the original vanilla villagers, pillagers and witches like from the start of the game the agents would become them but then I made it something like the agents will become them while living alongside them (cause the previous one contradicted the cool "Genesis" system , how could i even miss it , it was all ChatGPT's fault) and they will also have a dedicated trade window and they can trade using any items and trade any items in their inventory and well there was a central economic system that would decide the value of everything using the concept of supply and demand within the group of the agents which are living together to power the economy. They also had many different kind of jobs you can see it from [[NPC Professions in the RPG DivineWorld| here]] . Then for more lore I gave all these agent capabilities to the piglins as well and well since they were in nether so i needed to do something about it since they wont spawn unless a player goes to the nether and start the spawning of these mobs but oh well ChatGPT did something to fix that and initially we made it something like when the player or any agent visits the nether the piglins that spawns and then is near to the player the agent capabilities discussed so far will be activated for them and they will become agents as well and then even if the player or the agent leaves now since the piglin/s are now agents they will keep the nether loaded and now any piglin/s that comes near it/them (the piglin zombiefication thing was also there in which if zombified they can't be cured and their inventories will vanish except the stuff in both of their hands and will follow the mechanics after turning to zombie as discussed later and will follow the death mechanics of dying from lava in nether as discussed later) will have their agent capabilities activated originally this was also for the original gods (except oracle) as well it started when they were discovered but well lore would not be rich if that was the case so we and well during that time we were having some difficulties with cartographers,traders etc and nomads who would be stuck in the unloaded chunks of the minecraft world so well we did something since we were giving the agents player like capabilities we extended them from the Player class of the java so that they will be rendered properly and the world will properly render even if they go to unrendered regions (this was when the entire thing was in java once i move to python-java combo the problem resolved because I gave each of the agents their dedicated minecraft clients). Otherwise I and ChatGPT before had made something like tags for such professions so that they can traverse and load the unrendered regions. Oh yeah I had almost forgot that there was something like coming of age ceremony for the agents as well which would be tiny as children but would gradually increase in size by increase in scale of their body gradually to 1 when they would be considered as adults and there would be this ceremony. There was also something like if any normal agent eats a chorus fruit they will be able to teleport like endermen and will even have the ability to teleport cross dimentions and well be able to teleport entities within 10 block radius as well but the catch was that after 10 minecraft days they will turn into enderman permanently and the no. of times they will be able to teleport was directly proportional to the amount of chorus fruits eaten but if an agent eats a total of 10 chorus fruits in its lifetime then after eating the 10th fruit it will turn into an enderman forever (though this effect was limited to normal agents only for the other agents the powers will fade away by reducing power range gradually and 10th day it will vanish away). There was also an interesting death mechanic which I had designed whenever the agents were killed by a zombie attack in the overworld there was a small chance that the agent will become a zombie or husk depending on the location it will also follow the zombie -> drowned transformation logic and will hold the last held items on both hands (while keeping its inventory otherwise inventory would simply vanish with the corpse of the agent which remains for 3 minecraft days during which the agent can turn into its zombie after its death from a zombie attack) if it is cured within 30 minecraft days (using the vanilla minecraft's curing system for curing villagers), then the agent will have no memory of  its entire life, if not then after 30 days it'll turn into a skeleton or wither skeleton or stray depending on the location and will also follow the skeleton -> stray transformation logic in minecraft but when transforming into skeleton it will keep holding the items in its hands but will drop everything that was in its inventory. But if the agent died from lava in nether and but was somehow able to get to land before dying there was a chance it'll turn into a wither skeleton if it died from lava without being able to get on solid land then it would have a chance to become a ghast or blaze (blaze part added now). Oh yeah the gods also had the ability to turn invisible (all of them including the player). I guess towards the end of the java version DivineWorld I had also made DivineWorld to be self sustained and self updating , keeping a backup before updating and if update was stable oracle would allow it to update didn't really know what or how it did that just told ChatGPT to do it so that it can fix its edge cases and errors by itself and their was also something like multi-dimension (like dimension dimension not overworld-netherworld-end dimension something like parallel dimension) thing which the agents could travel to using different portals which could be created by the player god in which he can run a snapshot of the current world in it which would create a world similar to the original world where if needed things could be tested or well left alone to exist parrallely with the original world and well more than 1 such worlds could be made. Which could be copied off to make a different game altogether by using the snapshot of the world (Yeah I don't know what I was on back then). It was also a time when the oracle also kept watch on the entire functioning of the DivineWorld if something went wrong the oracle icon at the top right corner of the player god's screen would blink and when pressed it will spawn in front of the player and when tapped with an empty book it the oracle with write the problem (any problem) in that book. The problems would include the sight of an edge case , any agent that wasn't working properly, any system not working properly, any problems within the agents like diplomatic problems, economic problems, any crisis, etc.There was also a ritual for summoning the oracle which I don't really remember which was for the agents to spawn it so that they could ask it for advice on anything. The agents also had a relationship, breakup and marriage systems and gossip systems and  reputation systems as well. There was also something like a population cap of the agents which can be set to true or false if true default cap was 1000 which could be increased to 1000000 or decreased to 200.There was also something like an era system in DivineWorld like in prehistoric age the agents will use wood tools, in the stone age era they will use stone tools in the metal age they will use iron or copper or gold tools and in the modern era they would be able to use all kinds of tools and can also use redstone as well. They obviously had the ability to learn new things which helped in them getting a profession.Well in the middle version after the Chapters described above and before the current version there a way for the agents to communicate via chat bubbles appearing top of an agent head which was replaced with the current system which is more efficient. In the early verisons in Chapter 1 and Chapter 2 there was also something like a mode selection part there were 3 modes which were developed dev/debug mode, simulation mode, and divine mode. So in the dev/debug mode i was able to see a menu which when expanded showed me all of the internal important details of the agents which i could edit as well which would help me understand and debug the agents properly. I had also developed a Blockbench like thing integration in the mod so that the mod had a in built in blockbench like thing which had behaviors list that i could drop in there was also a sound list as well and well the option to add sound and i could also animate it basically all of the stuff of blockbench and where i can design and develop new mobs as well it would also have an AI assistant which will help me develop this mob I could also create behaviors for the new mob by describing the behavior to the AI assistant in plain english and it would use the specific Java codes to create that behavior for me which I can drop in after which I could add the Mob to the world. Well in the earlier phases like very early the start of Chapter 1 I was extremely worried about ownership for which I don't even remember all of the things I did. For timestamping my idea, so that later nobody could claim that they invented a system like this and I was just inspired by him/her, I did many things like going to a timestamping website I don't even remember the name but ChatGPT said that even companies timestamp their ideas there and created the [divine-world-dev](https://github.com/Architect-devlord/divine-world-dev) github repo solely for timestamp I also created a [Youtube teaser](https://www.youtube.com/watch?v=qC07ks1UO7M) for it as well just after a week of getting the idea of creating this. But still I was not satisfied what if someone else comes after my work and says he created it just that his contribution wasn't logged so I made everything in here all of it from the core ideas of the codebase to the blockbench models and also made a [Authorization.txt](https://github.com/Architect-devlord/divine-world-core/blob/main/Authorization.txt) in the divine-world-core repo yeah the code is AI generated but let me tell you how I even created the base I told the AI a problem which I am facing to reach a goal, mind you AI in the 2025 before December were quite dumb, the AI told me it was impossible that even big research institutions are trying to find an answer for that but me having the big brain I have I came up with a big revolutionary solution that will change the very course of the entire AI research industry but **OH WELL THE AI REMEMBER'S THAT THAT THING WAS ALREADY BEING USED IN EXPERIMENTS IN RESEARCH LABS AND ARE ALREADY PUBLISHED. WHY WAS IT PUBLISHED AFTER I CAME UP WITH UNLESS......BINGO IT ALREADY WAS THERE THE AI WAS JUST STOOBID. MAN WAS IT SO HORRENDOUS WHEN I FOUND THAT OUT**. But oh well even the robot designs are mine. And for this I had created DivineWorld originally as a paper plugin so that the codebase doesn't even touch the client or visiter's computer so that it couldn't be recieved by the client computer and therefore there was no chance of it being decompiled into source code additionally I had something like a liscence key generator to validate only selected persons to be able to run DivineWorld you could get this key only through me and once used its impossible to reuse it again. The wish to protect my codebase was so much that  I decided to create a virus that would reveal the location of anyone's computer if they are trying to decompile it while also melting their computer just after that (could've use a zip bomb for it ), but well ChatGPT disaggreed so I had no way of creating it. But oh well at the end I moved to forge for the sheer versatility of it even at the risk of exposing my codebase cause I decided I won't keep it closed source for long and open source it after a very important event. 

EDIT:
I am shifting to neoforge from forge to support newer versions of minecraft (26.x).

Well you can check the eventual fate of the old version of the RPG style DivineWorld in [[raw/index|raw]] files.
And well the current fate here in the [[index | wiki]] files.