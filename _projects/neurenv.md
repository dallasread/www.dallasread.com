---
layout: principle
title: Neurenv
description: An exploration in neural nets and how they evolve.
---

Check the developer console. 👻

<script type="text/javascript">
  const CLEAR_SCREEN = '\033[2J'
  const shuffle = (arr, n) => arr.sort(() => 0.5 - Math.random()).slice(0, n)
  const rand = (min, max) => Math.random() * (max - min) + min
  const randInt = (min, max) => parseInt(rand(min, max))

  class Neuron {
    constructor(name) {
      this.name = name
    }

    nameWithType() {
      return `${this.constructor.name}:${this.name}`
    }
  }

  class Sensor extends Neuron { }

  class SightSensor extends Sensor { }

  class Urge extends Neuron { }

  class Synapse {
    constructor(from, to, weight, activation) {
      this.from = from
      this.to = to
      this.weight = weight
      this.activation = activation
    }
  }

  class Brain {
    constructor(creature, brainPower) {
      this.creature = creature

      this.sensors = [
        new Sensor('SEE_NORTH'),
        new Sensor('SEE_SOUTH'),
        new Sensor('SEE_EAST'),
        new Sensor('SEE_WEST'),
      ]

      this.innerNeurons = Array.from(Array(brainPower).keys(), (index) => new Neuron(`INNER_${index}`))

      this.urges = [
        new Urge('MOVE_NORTH'),
        new Urge('MOVE_SOUTH'),
        new Urge('MOVE_EAST'),
        new Urge('MOVE_WEST'),
      ]

      this.synapses = []

      this.wireRandomly()
    }

    wireRandomly() {
      shuffle(this.sensors, randInt(1, this.sensors.length))
        .forEach((sensor) => {
          shuffle(this.innerNeurons, randInt(1, this.innerNeurons.length))
            .forEach((innerNeuron) => this.addSynapse(sensor, innerNeuron))
        })

      shuffle(this.innerNeurons, randInt(0, this.innerNeurons.length))
        .forEach((innerNeuronFrom) => {
          shuffle(this.innerNeurons, randInt(0, this.innerNeurons.length))
            .filter((n) => n !== innerNeuronFrom)
            .forEach((innerNeuronTo) => this.addSynapse(innerNeuronFrom, innerNeuronTo))
        })

      shuffle(this.innerNeurons, randInt(1, this.innerNeurons.length))
        .forEach((innerNeuron) => {
          shuffle(this.urges, randInt(1, this.urges.length))
            .forEach((action) => this.addSynapse(innerNeuron, action))
        })
    }

    addSynapse(from, to) {
      this.synapses.push(new Synapse(from, to, Math.random(), Math.random()))
    }

    execute() {
      const urgeWeights = Array.from(Array(this.urges).keys(), (index) => 0)

      this.sensors.forEach(() => {

      })

      return urgeWeights
    }

    log() {
      console.log(
        this.synapses.map((s) => {
          return [s.from.nameWithType(), s.weight.toFixed(1), s.to.nameWithType()].join(' -> ')
        }).join('\n')
      )
    }
  }

  class Creature {
    constructor(world, brainPower) {
      this.world = world
      this.brain = new Brain(this, brainPower)
      this.isDestroyed = false
      this.position = {
        x: randInt(0, this.world.width),
        y: randInt(0, this.world.height),
      }
    }

    adapt() {
      if (this.isDestroyed) {
        return
      }

      this.act( this.brain.execute() )
    }

    act(urgeWeights) {

    }

    symbol() {
      return this.isDestroyed ? 'x' : '*'
    }

    destroy() {
      this.isDestroyed = true
    }
  }

  class World {
    constructor(mapWidth, mapHeight, numberOfCreatures, brainPower) {
      this.width = mapWidth
      this.height = mapHeight
      this.brainPower = brainPower
      this.creatures = Array.from(Array(numberOfCreatures).keys(), () => new Creature(this, brainPower))
    }

    loop(n, delay) {
      this.creatures.forEach((c) => c.adapt())
      this.render()

      if (n <= 0) return

      setTimeout(() => this.loop(n - 1, delay), delay)
    }

    renderCreature(x, y) {
      const creature = this.creatures
        .find((c) => c.position.x === x && c.position.y === y)

      return creature ? creature.symbol() : ' '
    }

    render() {
      const header = Array.from(Array(this.width + 5).keys(), () => '-').join('')
      const rows = []

      for (var y = 0; y <= this.height; y++) {
        const row = []

        for (var x = 0; x <= this.width; x++) {
          row.push(this.renderCreature(x, this.height - y))
        }

        rows.push('| ' + row.join('') + ' |')
      }

      console.log(header + '\n' + rows.join('\n') + '\n' + header)
      console.log('REMAINING: ' + this.creatures.filter((c) => !c.isDestroyed).length)
    }
  }

  const NUMBER_OF_CREATURES = 20
  const MAP_WIDTH = 60
  const MAP_HEIGHT = 15
  const BRAIN_POWER = 4
  const ITERATIONS = 1000
  const ITERATION_DELAY = 100

  const world = new World(MAP_WIDTH, MAP_HEIGHT, NUMBER_OF_CREATURES, BRAIN_POWER)
  world.creatures.forEach((c) => c.brain.log())
  // world.loop(ITERATIONS, ITERATION_DELAY)
  world.render()
</script>