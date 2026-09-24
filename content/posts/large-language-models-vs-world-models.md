+++
title = "Large Language Models vs World Models: A Simple Guide"
description = "A plain-English guide to large language models and world models: how each one learns, where each one breaks, and what people use them for, with real examples from ChatGPT, Genie 3, V-JEPA 2, Dreamer, and Waymo."
date = 2026-09-24

[taxonomies]
tags = ["ai", "llm", "world-models", "explainers"]

[extra]
toc = true
comment = false
+++

Push a glass of water toward the edge of a table and you know how the scene ends before it ends. The glass tips, the water spills, and the glass probably breaks. You never memorized that as a rule. Your brain ran a quick simulation and showed you the result.

Ask a chatbot what happens next and you get a good answer too: the glass falls, it likely shatters, and the water spreads across the floor. The chatbot got there by a different route. Most of what it knows comes from text, including plenty of sentences written by people who had watched glasses fall.

The chatbot is a large language model. Your brain, in this example, is doing the job of a world model.

> A **large language model** learns about the world from what people wrote, and predicts the next word. A **world model** learns from the world itself, through video, sensor data, and recorded actions, and predicts what happens next when something acts.

Chatbots built on the first kind have been in wide use since ChatGPT launched in late 2022. The second kind is newer to the headlines. In February 2026, Waymo said it generates rare driving scenes, such as a tornado ahead on the road, with a world model built on Google DeepMind's Genie 3. In March 2026, Yann LeCun, Meta's former chief AI scientist, [raised $1.03 billion](https://www.reuters.com/business/ex-meta-ai-chief-yann-lecuns-ami-raises-103-billion-alternative-ai-approach-2026-03-10/) for AMI Labs, a startup built around world models. He told Reuters that AI built on predicting the next word or pixel will not produce broadly capable agents by itself.

## Two kinds of prediction

Both kinds of model do the same basic job: they predict what comes next. They differ in what "next" means.

```
Large language model
  input:   the text so far
  output:  the next word

World model
  input:   the world right now, plus an action
  output:  the world a moment later
```

For an LLM, a word is usually a token: a whole word or a piece of one. For a world model, the world can be a video frame, a set of sensor readings, a game screen, or a compact description of a scene that only the model can read. An action can be a step forward, a turn of the steering wheel, a command to a robot arm, or a key press in a game.

## What is a large language model?

### The training game: guess the next word

Take a sentence from a book, "The cat sat on the ___," and hide the last word. A program guesses "floor." The real word was "mat." The program nudges its internal numbers so that "mat" looks a little more likely next time.

Repeat that across an enormous pile of web pages, books, and code, and the program gets very good at continuing almost any kind of writing. That program is a large language model, or LLM. The pile is big: Meta trained [Llama 3.1 405B](https://ai.meta.com/blog/meta-llama-3-1/) on over 15 trillion tokens.

The internal numbers are called parameters, and "large" refers to how many there are. [GPT-3](https://arxiv.org/abs/2005.14165), which OpenAI described in 2020, had 175 billion. That Llama model has 405 billion. The biggest labs no longer publish the count for their flagship models.

### Guessing is harder than it sounds

To finish a line of Python, the model has to follow what the code does. To continue a mystery novel, it has to keep track of who was in which room. Training rewards anything that improves the guess, so the model picks up facts and patterns of reasoning along the way.

### From autocomplete to assistant

A model trained only on next-word guessing is a strong autocomplete. Give it a question, though, and it may reply with more questions, because on the internet a question is often followed by another question.

The companies that build LLMs add a second round of training. People write examples of good answers. People also compare pairs of the model's answers and mark the better one, and the model learns from those choices. That second part is called reinforcement learning from human feedback, or RLHF. OpenAI described the method in its 2022 [InstructGPT paper](https://arxiv.org/abs/2203.02155) and used it for ChatGPT.

### The engine underneath

Nearly every LLM today is built on the transformer, a way of wiring up the model that Google researchers introduced in the 2017 paper ["Attention Is All You Need"](https://arxiv.org/abs/1706.03762). Its central trick is attention. While working on one word, the model can look back at every earlier word and weigh which ones matter. In "The trophy did not fit in the suitcase because it was too big," attention is how the model connects "it" to "trophy."

### Examples you know

ChatGPT (OpenAI), Claude (Anthropic), Gemini (Google), Llama (Meta), DeepSeek, Qwen (Alibaba), and Mistral all run on LLMs. Most of them now accept images and audio as well as text, so the name "language model" undersells them a little. Text is still the core of how they learned.

### The weak spots

An LLM is trained to produce likely text. Likely text is usually true, but the model has no separate step that checks. If it doesn't know something, it can still produce a fluent and confident wrong answer. People call this hallucination.

In 2023, two New York lawyers filed a brief that cited six earlier court decisions. ChatGPT had invented all six. One of the lawyers had asked ChatGPT whether the cases were real, and it said yes. A federal judge in Manhattan [sanctioned the lawyers](https://www.reuters.com/legal/new-york-lawyers-sanctioned-using-fake-chatgpt-cases-legal-brief-2023-06-22/) and their firm with a $5,000 fine.

Two more limits matter for this comparison. First, an LLM's picture of the physical world is secondhand. People rarely write down the obvious. You won't find many posts that say "I set the cup down and it stayed where I put it," even though that happens every time. Second, an LLM can't test anything. It can't push the glass and watch. If its idea of how something works is wrong, nothing in training catches the error as long as the next word comes out right.

## What is a world model?

### An old idea

In 1943, the Scottish psychologist Kenneth Craik argued that a mind carries a "small-scale model" of the world and of its own possible actions. With that model, he wrote, a person can "try out various alternatives, conclude which is the best of them, react to future situations before they arise." AI researchers still quote that passage when they explain what a world model is for.

You use one all day. David Ha and Jürgen Schmidhuber, whose 2018 paper helped popularize the term in AI, describe a baseball batter. The batter has milliseconds to decide how to swing, less time than the signal from the eyes takes to reach the brain. Batters still hit 100 mph fastballs, because the brain predicts where the ball will be and sends the bat there.

### The AI version

In AI, a world model is a program that learns how an environment changes. It answers one question over and over: the environment is in this state, this action happens, which state comes next?

The environment is whatever the model learns about, such as a video game or a city street. A state is a snapshot of it at one moment. An action is something done to it, like pressing jump or turning the wheel.

A world model learns from recordings where the actions are known, such as gameplay with the button presses logged or robot footage with the motor commands. Once trained, it can run forward by itself. An AI agent, meaning a program that takes actions toward a goal, can try an action inside the world model, see the predicted result, and then decide whether to do it for real.

A video generator makes a clip that plays from start to finish. A world model waits for the next action and shows what that action causes.

### Learning inside a dream

Ha and Schmidhuber's [2018 paper](https://worldmodels.github.io/) showed the idea working. Their agent had three parts: one compressed each game frame into a short summary, one predicted the next summary, and a tiny controller chose actions. The first two parts formed the world model.

They trained the world model on a stripped-down Doom level built for AI research, where the player dodges fireballs. Then they trained the controller entirely inside the world model's own simulated version of the level, which they called a dream. Only after that did they put the controller in the real game. It survived about 1,100 time steps on average, well past the 750 that count as solving the level.

The dream had a flaw, and the controller found it. In some runs it learned to move in a way that stopped the dream's monsters from firing at all. Monsters in the real game don't cooperate like that. The researchers fixed it by making the dream noisier, so the controller couldn't count on the glitch. The lesson still holds: an agent trained inside a world model will exploit whatever the model gets wrong.

### One name, three meanings

Headlines use "world model" for three different things:

| What people mean | Examples |
| --- | --- |
| A simulator you can step into: type a prompt, get a world you can move through | Genie 3, Marble |
| A planning engine inside a robot or game agent that predicts what each possible action will do | V-JEPA 2, Dreamer |
| The rough picture of the world that an LLM may build inside itself while learning to predict text | The board-game and map studies [further down](#does-an-llm-have-a-world-model-inside-it) |

### Drawing the future or summarizing it

World models also differ in what they output.

Some draw the future pixel by pixel. Genie 3 generates each video frame from the frames before it and your latest input. NVIDIA's Cosmos models generate video too. People can watch the output, which makes these models useful for producing training footage and test scenes.

Others predict a compact summary of what comes next and never draw it. Meta's V-JEPA 2 works this way. It follows an approach Yann LeCun argued for at Meta: much of a scene can't be predicted and doesn't matter for the task at hand. A robot carrying a cup across a room doesn't need to predict how each leaf on the tree outside the window will move.

In one test, skipping the pixels paid off in speed. Meta's team compared the two styles on the same robot. Planning with Cosmos, which draws video, took 4 minutes for each arm movement, so moving one object took over an hour. V-JEPA 2 needed 16 seconds per movement and succeeded more often.

## World models you can look at today

| Model | Made by | What it does | Released |
| --- | --- | --- | --- |
| [Genie 3](https://deepmind.google/blog/genie-3-a-new-frontier-for-world-models/) | Google DeepMind | Turns a text prompt into a world you can walk through in real time | Announced August 2025 |
| [V-JEPA 2](https://ai.meta.com/blog/v-jepa-2-world-model-benchmarks/) | Meta | Learned from over a million hours of video; helps robot arms plan their moves | June 2025, free to download |
| [Cosmos](https://nvidianews.nvidia.com/news/nvidia-launches-cosmos-world-foundation-model-platform-to-accelerate-physical-ai-development) | NVIDIA | Makes realistic video for training robots and self-driving cars; learned from 20 million hours of video | January 2025, free to download |
| [Marble](https://www.worldlabs.ai/blog/marble-world-model) | World Labs | Builds 3D worlds from text, photos, or video that you can save and reuse in games and 3D tools | November 2025 |
| [Waymo World Model](https://waymo.com/blog/2026/02/the-waymo-world-model-a-new-frontier-for-autonomous-driving-simulation/) | Waymo, built on Genie 3 | Makes rare driving scenes to test Waymo's cars, as camera images and lidar (laser distance sensor) data | February 2026 |
| [Dreamer 4](https://arxiv.org/abs/2509.24527) | Google DeepMind | Learned Minecraft from recorded play and practiced only inside its own simulation | September 2025, research paper |

Google has also opened a prototype called [Project Genie](https://blog.google/innovation-and-ai/models-and-research/google-deepmind/project-genie/) to Google AI Ultra subscribers in the US. Each generated world runs for up to 60 seconds.

## The differences, side by side

| | Large language model | World model |
| --- | --- | --- |
| Predicts | The next word | What happens next in a scene |
| Learns from | Text and code, plus images and audio in newer models | Video, sensor data, and game or robot recordings, ideally with the actions taken |
| Role of actions | None in the main training stage | Central: "if this happens, what comes next?" |
| Output you see | Words and code | Video frames, 3D scenes, or predictions a robot uses |
| Strong at | Language, knowledge, writing, coding | Space, motion, cause and effect, planning physical moves |
| Typical failure | Makes up facts with confidence | Physics glitches; worlds that drift over time |
| How widely used | ChatGPT alone reached about 10% of the world's adults by mid-2025 | Mostly research and early products |

Three differences in that table do most of the work.

The biggest difference is the role of actions. An LLM can tell you what usually happens when someone knocks a glass off a table, because people have written about it. A world model is built to predict what happens when this particular glass gets this particular push, including situations nobody ever wrote down.

The training data differs. Text is plentiful, and it already describes the world in words. Video is plentiful too, but most of it doesn't record the actions behind it. A dashcam clip shows the road but not how the driver turned the wheel. Video with the actions recorded is the scarce ingredient for world models, so builders combine a lot of plain video with a little of the rare kind. V-JEPA 2 learned mostly from ordinary video and needed under 62 hours of robot footage with the actions recorded.

The feedback differs. During training, an LLM is graded on whether it guessed the next word of a document. A world model trained on video is graded on whether its predicted frame matches the frame the camera recorded.

## Same job, two approaches

### Making a game

Ask an LLM for a small browser game and it writes code. The result is precise. The jump height is a number, the walls stay where they are, and the game plays the same way tomorrow.

Ask Genie 3 for a game world and it draws the world frame by frame while you move through it. You can type "a snowy mountain village at dusk" and walk around in it, with no code at all. The world does not last long, though. Genie 3 keeps a world consistent for a few minutes and remembers changes you make for up to a minute.

### Moving a cup with a robot arm

Give V-JEPA 2 a photo of how the table should look at the end, say with the cup inside the bowl. The robot then plans in a loop:

```
goal photo: cup inside the bowl

1. list a few possible arm movements
2. predict the result of each one with the world model
3. pick the movement whose predicted result is closest to the goal
4. make that one movement for real, look again, repeat
```

Meta ran this on robot arms in two labs, without collecting any data from those robots and without training for the specific task. Averaged over the two labs, the arms picked up a cup and put it in the right place 80 percent of the time, and did the same with a box 65 percent of the time.

An LLM can write a sensible list of steps for the same job: open the gripper, move above the cup, lower, close, lift. It has no reliable way to tell whether moving three centimeters to the left will knock the cup over. That question is about this cup on this table, and answering it takes a model of the physical scene.

### Mining diamonds in Minecraft

Getting a diamond in Minecraft takes a long chain of steps: chop wood, craft tools, mine stone, find iron, smelt it, craft an iron pickaxe, and dig deep enough to find diamonds. The Dreamer 4 team at Google DeepMind set a harder version of the challenge: learn only from about 2,500 hours of recorded human gameplay, with no chance to play the game during training.

Dreamer 4 learned a world model of Minecraft from those recordings, then practiced inside it. It became the first agent to get diamonds from recorded data alone. It reached a diamond in 0.7 percent of its hour-long test runs. That rate is low, and the task takes more than 20,000 mouse and keyboard actions to finish.

The team also built a second agent on top of Gemma 3, a Google model that understands images and text, the same kind of model that powers chatbots. Dreamer 4 crafted iron pickaxes nearly three times as often as that agent.

## Does an LLM have a world model inside it?

This question matters for the whole comparison. If LLMs already carry an accurate picture of the world, separate world models add less. To find out, researchers train language-style models on worlds they understand completely, like a board game, a street map, or a solar system. Then they check whether the model learned the world itself or only patterns that happen to fit the training data. The results point both ways.

### Evidence for: a board it never saw

Othello is a board game played on an 8-by-8 grid. In a [2023 study](https://arxiv.org/abs/2210.13382), researchers trained a small language model on nothing but lists of Othello moves. It never saw a board or the rules, yet it learned to suggest legal moves.

When the researchers looked inside the model, they found a picture of the board: they could read off which squares held which pieces. When they changed that inner board, as if moving a piece by hand, the model's suggested moves changed to match. Keeping track of the board helped the model predict moves, so it learned to keep track of the board.

### Evidence against: a map that doesn't exist

In a [2024 study](https://arxiv.org/abs/2406.03689), researchers from Harvard, MIT, and Cornell trained language-style models on New York City taxi trips, written out as turn-by-turn directions. The models got very good at giving directions: 96 to 99 percent of their routes were valid.

Then the researchers drew the street map that the models' answers implied. It was full of streets that don't exist, curving across Manhattan and linking intersections far apart.

The fake streets mattered as soon as the city changed. Take the model trained on the shortest routes. It produced valid routes 99 percent of the time. When the researchers forced a detour at 1 turn in 100, that dropped to 69 percent. At 1 turn in 10, it dropped to 8 percent.

A model trained on aimless random drives, the kind no real taxi takes, held up far better. It still found valid routes 97 percent of the time with half the turns detoured, because it had seen more of the city. The researchers' stricter checks still found flaws in its map.

### Evidence against: orbits without gravity

In a [2025 study](https://arxiv.org/abs/2507.06952), a team from Harvard and MIT trained a model on the movements of planets in 10 million made-up solar systems. It learned to predict where each planet would go next with high accuracy.

Then the researchers asked it about the force behind those orbits: the pull of gravity on each planet. They worked out the rule the model was following, and it made no physical sense. The rule also changed from one group of solar systems to the next.

In the early 1600s, Johannes Kepler described the shapes of planetary orbits. In 1687, Isaac Newton explained those shapes with a law of gravity. The model had learned something like Kepler's shapes without Newton's law.

The team ran a smaller version of the test on three well-known LLMs: OpenAI's o3, Anthropic's Claude Sonnet 4, and Google's Gemini 2.5 Pro. Newton's law says the pull depends on both masses and on the distance between them. The rules behind all three models' answers left the distance out, and each used only one of the two masses. Any of these models will recite Newton's law if you ask.

### A twist from 2026

A [follow-up in 2026](https://arxiv.org/abs/2602.06923) by another team showed a way around the problem. When the researchers let the model look only at a planet's last two positions instead of its whole history, the model worked out the pull of gravity. With the whole history in view, it went back to tracing ellipses the way Kepler did.

### The short answer: partly

LLMs build pieces of a world model as a side effect of predicting text, and some of those pieces are good. Training only checks whether the pieces help predict the next word. They can be right in common situations and wrong in unusual ones, with no warning sign. World-model research puts the model of the world first and tests it against what happens.

## Use cases

### LLMs

A September 2025 [study of ChatGPT use](https://www.nber.org/papers/w34255), by OpenAI researchers and Harvard economist David Deming, found that three kinds of conversation made up nearly 80 percent of it: practical advice, looking things up, and writing. More than 70 percent of messages had nothing to do with work.

Common uses, with an example of each:

| Use | Example |
| --- | --- |
| Writing and editing | Turning rough notes into a clear email |
| Answering questions | Explaining a term from a medical report in plain words |
| Coding | Writing a function, explaining an error, reviewing a pull request |
| Working with documents | Summarizing a long contract, pulling totals out of invoices |
| Translation | Turning a help article into Hindi or Spanish |
| Agents | Fixing a bug across several files by running tools, one step at a time |

Coding is a small slice of ChatGPT traffic, but it is the whole job of dedicated tools like GitHub Copilot, Cursor, and Claude Code.

### World models

Robots need practice, and practice in the real world is slow and hard on the hardware. World models give robots somewhere else to practice.

- NVIDIA offers Cosmos as a way to make training video for robots. At the Cosmos launch, Agility, which builds humanoid robots, said the model lets it create training scenes and rely less on expensive real-world recording.
- V-JEPA 2 plans robot-arm moves, as in the cup example above.
- Marble builds rooms for robot simulators. In a [World Labs case study](https://www.worldlabs.ai/case-studies/1-robotics), a contributor to the RoboSuite simulator said one high-quality simulation environment can take weeks to put together by hand. With Marble, several usable environments took hours.

Self-driving cars face a different data problem. The dangerous moments are rare, so a fleet can drive for a very long time without meeting a tornado or an elephant in the road. Waymo's world model generates those scenes on demand, and engineers can change a scene with a plain-language prompt. In February 2026, Waymo said its cars had driven nearly 200 million fully autonomous miles on real roads and billions of miles in simulation. NVIDIA also named Uber and Waabi among the first companies to adopt Cosmos.

Training AI agents is the use DeepMind emphasizes for Genie 3. The team gave its SIMA agent goals inside worlds that Genie 3 generated. SIMA chose the moves, and Genie 3 drew what happened, simulating the results of each move without knowing the goal.

Games, film, and design are the most visible uses. Project Genie lets people explore worlds they describe. World Labs shot its Marble launch video on an LED stage with Marble worlds as the backdrops, and a game developer built a rail shooter in Unreal Engine 5 from Marble scenes.

## They work best together

The two kinds of model cover each other's gaps. An LLM is good at understanding what you want and breaking it into steps. A world model is good at predicting what a physical step will do. Several real systems already pair them:

- [SIMA 2](https://deepmind.google/blog/sima-2-an-agent-that-plays-reasons-and-learns-with-you-in-virtual-3d-worlds/), the next version of that agent, uses Gemini to understand instructions and reason about goals. DeepMind reports that it kept improving through its own practice in Genie 3 worlds it had never seen.
- Project Genie uses Gemini and Google's Nano Banana Pro image model to help you describe and preview a world before Genie 3 generates it.
- Meta connected V-JEPA 2 to a language model so the pair can answer questions about videos.

Picture a home robot asked to put a mug in the dishwasher. A language model turns the request into steps: find the mug, open the dishwasher, place the mug on the top rack. For each step, a world model tries a few possible arm movements in simulation and picks the one that ends with the mug on the rack instead of the floor.

## The limits of world models today

World models are early, and their builders are open about the limits. DeepMind's own list for Genie 3:

- Worlds hold together for a few minutes of continuous interaction.
- Agents can take only a limited range of actions inside it.
- Several characters acting on their own in one shared world is still an unsolved research problem.
- It can't recreate real places with perfect accuracy.
- It tends to render readable text only when that text appears in the prompt.

Two broader problems apply across the field. Agents trained inside a world model exploit its mistakes, as the 2018 Doom agent did with the fireballs. And a model can look right while being wrong. A generated video can look real and still break physics, and the taxi and orbit studies show that accurate predictions can hide a wrong model underneath. The test that counts is whether plans made inside the model work in the real world.

## A quick test for AI headlines

Three questions sort most announcements.

1. What does the model predict? The next word points to an LLM. The next frame, sensor reading, or state points to a world model.
2. Can something act inside it and see the consequences? If a person or an agent can take actions and the model responds to each one, it behaves like a world model. A video generator that plays one clip from one prompt does not meet that bar.
3. How was it checked? Look for tests against real outcomes, such as a robot that finished the task or a plan that worked outside the simulation. A realistic-looking video, on its own, says little about the physics inside the model.

## Further reading

- Google DeepMind, [Genie 3: A new frontier for world models](https://deepmind.google/blog/genie-3-a-new-frontier-for-world-models/)
- Meta AI, [Introducing the V-JEPA 2 world model](https://ai.meta.com/blog/v-jepa-2-world-model-benchmarks/), and the [V-JEPA 2 paper](https://arxiv.org/abs/2506.09985)
- David Ha and Jürgen Schmidhuber, [World Models](https://worldmodels.github.io/) (2018)
- Danijar Hafner and colleagues, [Training Agents Inside of Scalable World Models](https://arxiv.org/abs/2509.24527) (Dreamer 4)
- Keyon Vafa and colleagues, [Evaluating the World Model Implicit in a Generative Model](https://arxiv.org/abs/2406.03689) (the taxi map)
- Keyon Vafa and colleagues, [What Has a Foundation Model Found?](https://arxiv.org/abs/2507.06952) (the orbits)
- Ziming Liu and colleagues, [From Kepler to Newton](https://arxiv.org/abs/2602.06923)
- Kenneth Li and colleagues, [Emergent World Representations](https://arxiv.org/abs/2210.13382) (Othello-GPT), with a readable summary on [The Gradient](https://thegradient.pub/othello/)
- Waymo, [The Waymo World Model](https://waymo.com/blog/2026/02/the-waymo-world-model-a-new-frontier-for-autonomous-driving-simulation/)
- NVIDIA, [Cosmos launch announcement](https://nvidianews.nvidia.com/news/nvidia-launches-cosmos-world-foundation-model-platform-to-accelerate-physical-ai-development)
- World Labs, [Marble: A Multimodal World Model](https://www.worldlabs.ai/blog/marble-world-model)
- Aaron Chatterji and colleagues, [How People Use ChatGPT](https://www.nber.org/papers/w34255) (NBER, 2025)
- Ashish Vaswani and colleagues, [Attention Is All You Need](https://arxiv.org/abs/1706.03762) (2017)
