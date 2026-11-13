---
title: Tools
permalink: /tools/
layout: page
wide: true
excerpt: Offensive security tools built by 7heAnsw3r — targeting x86_64 and ARM architectures.
comments: false
---

<p class="pg-lead">Tools I'm building for penetration testing, malware development research and CTF work. Everything is designed to run on both <code>x86_64</code> and <code>ARM</code>. Published as they reach a stable state.</p>

<div class="tool-grid">

  <article class="tool-card cat-maldev">
    <div class="tool-top"><span class="tool-status">in development</span><span class="tool-lang">C</span></div>
    <h3>Cross-Architecture Implant Framework</h3>
    <p>Lightweight C2 implant targeting x86_64 and ARM (including embedded/IoT targets). Focused on minimal footprint and evasion of common EDR signatures.</p>
    <div class="chip-row"><span>x86_64</span><span>ARM</span><span>ARM64</span><span>AMSI bypass</span><span>ETW patching</span><span>process injection</span></div>
  </article>

  <article class="tool-card cat-ad">
    <div class="tool-top"><span class="tool-status">in development</span></div>
    <h3>AD Enumeration Toolkit</h3>
    <p>Fast enumeration modules for Active Directory environments — wraps common Impacket/BloodHound workflows into a single recon pipeline.</p>
    <div class="chip-row"><span>Impacket</span><span>BloodHound</span><span>recon</span></div>
  </article>

  <article class="tool-card cat-research">
    <div class="tool-top"><span class="tool-status">in development</span></div>
    <h3>Shellcode Utilities</h3>
    <p>Position-independent shellcode helpers for process injection chains — compatible with both Windows x86_64 and ARM64.</p>
    <div class="chip-row"><span>x86_64</span><span>ARM64</span><span>process injection</span></div>
  </article>

</div>

<a class="gh-banner" href="https://github.com/7heAnsw3r" target="_blank" rel="noopener">
  <span class="hl-prompt">$ git remote -v</span>
  <span>All public tools live at <strong>github.com/7heAnsw3r</strong> →</span>
</a>
