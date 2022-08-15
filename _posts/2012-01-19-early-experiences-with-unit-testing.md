---
title: "Early Experiences with Unit Testing"
layout: post
---



I got my feet wet with unit testing during a summer internship, before the term was officially on my radar. I was writing a number-crunching library in C, and my tests took the form of short C programs using that library. They would run through several related examples and print to the console when something went wrong. It's a shame that I was a senior in college before the concept even came up, but at least I got exposure to it before graduating and entering the workforce. This approach actually helped me to catch a pretty serious and subtle error that I wasn't explicitly looking for.

## The Goal

The project involved a lot of number crunching for an electron-detector that would be mounted on a satellite. Picture a 180 degree protractor stuck on the side of a spinning box. At time 0, we might get a reading of an electron coming from 15 degrees. At time 1 we might get another reading coming from 42 degrees.

These numbers on their own were completely meaningless, because the device was ignorant of the fact that it was mounted on the side of a spinning satellite. 15 degrees right now is a completely different direction from 15 degrees a moment from now. The idea is that we needed to 'rotate' the raw data backwards, against the angle the satellite would rotate through, to know what direction it really came from.

To complicate matters, the satellite would be orbiting the Earth, and in turn orbiting the Sun, introducing more angles to reverse-rotate. As fuel was consumed, a predictable wobble would be introduced, leading to several other rotations to reverse-rotate. Additionally, we needed to translate the resulting 3D coordinates in terms of a convenient coordinate system: some convienient origin (0, 0, 0) with well-defined X, Y, and Z axes. Different people needed to see the results according to different coordinate systems. Some wanted the origin to be the center of the Earth, with Z+ moving throught the magnetic north pole, some wanted the origin to be the center of the sun with Z+ perpendicular to the plane Earth moves through, etc.

The bottom line was that the physicists would determine exactly what rotations were involved, and how to calculate it all in terms of the initial start position of the satellite and a known time to start the clock. I just had to turn the crank and calculate what those rules dictated.

## The Short Cut

It turns out there's a neat short-cut when doing multiple rotations around 3D axes. It comes up in 3D graphics, too. There's a way to build a 3x3 matrix representing a single rotation around a single one of the 3 possible axes. If you multiply that matrix against a single 3D data point, you get a new data point showing where it had really come from. Each bit of the phsysicist's work could be turned into such a matrix, and if you multiplied all of these together you'd get yet another 3x3 matrix representing the *combination* of all the many rotations. Armed with this, you could very quickly process all the data points without having to do a lot of trig for each one individually. The goal was to have a 3x3 grid of numbers that bottled up *every* operation to perform.

## My First Unit Tests

Despite that simplification, **I was sure I was going to make mistakes**. I started writing very short programs to test my matrix-twiddling library as I went along. I had some concrete examples from the physicists and from the satellite's documentation to get me started. **Occasionally I'd be surprised by a result, and would know something was either wrong with my code or with my understanding of the problem.** Not doing so would have been unprofessional, and it would have surely produced a useless mess: a program that would serve only to make the machine warmer for the few minutes it ran.

**It was really rewarding to see how the testing effort motivated my approach to the implementation.** It forced me to start thinking in terms of the smaller building blocks. Once I was confident in the individual parts, I'd write the next set of tests at a higher level to test how those parts worked together. I generally hate Lego analogies for writing software, but there was a nearly-audible plasticky *click* whenever a new operation started passing its tests.

## The Payoff, or Why We Should All Be Right-Handed

Eventually this testing found a *very big* discrepancy between the explanation of the rotations involved, and the documentation of the satellite, even though both documents were intended as a matching set.

In middle school Algebra, we always draw +Y moving up the page, and +X moving to the right of the page. That's just a convention; it makes sense only because we're all agreeing that we will do it that way. When you introduce 3D, you have to decide whether the +Z axis is coming "up" out of the page, or "down" into the table the page sits on. One of these is called a left-handed system, and the other is called a right-handed system. Again, it's just convention; if everyone agrees on what we're using, everything is fine.

The satellite's examples used the opposite +Z from the documentation of the angles to reverse-rotate. Every time we needed to rotate around Z, we were going in the wrong direction!

**I never would have discovered this problem if I hadn't been building up a test-suite alongside the library.** It was just dumb luck that I got to experience a unit-testing "A-HA!" moment so long ago, but I'm glad I did.
