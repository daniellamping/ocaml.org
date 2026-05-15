---
title: Odoc bugs
description:
url: https://jon.recoil.org/blog/2025/09/odoc-bugs.html
date: 2025-09-22T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#odoc-bugs" class="anchor"></a>Odoc bugs</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2025-09-22</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/odoc.html" title="odoc">odoc</a></p></li></ul>
<ul class="at-tags"><li class="x-ocaml.requires"><span class="at-tag">x-ocaml.requires</span> <p>odoc.model</p></li></ul>
<p>This post is a brief write-up of a couple of bugs in odoc that I've been working on over the past 2 weeks. I was convinced at the start of this that I was actually fixing one bug, but although they both had the same backtrace and similar immediate causes, they're actually quite different. They both involve <em>expansion</em>, which is the process that odoc uses to work out the contents of a module from its expression - what allows you to see the contents of a module such as <code>module M = Map.Make(String)</code>.</p>
<h3><a href="https://jon.recoil.org/atom.xml#bug-930:-inline-destructive-substitutions" class="anchor"></a>Bug 930: inline destructive substitutions</h3>
<p>Bug #930 in odoc is about a substitution problem:</p>
<div><pre class="language-ocaml"><code>module type S1 = sig
  type t0
  type 'a t := unit

  val x : t0 t
end

module type S2 = sig
  type t (* must be the same name as [S1.t] *)

  include S1 with type t0 := t
end

module type S3 = sig
  type t1

  include S2 with type t := t1
end</code></pre></div>
<p>which when processed by odoc 2.4 throws an exception:</p>
<pre>odoc: internal error, uncaught exception:
      Invalid_argument("List.fold_left2")
      Raised at Stdlib.invalid_arg in file "stdlib.ml", line 33, characters 20-45
      Called from Odoc_xref2__Subst.type_expr in file "subst.ml", line 598, characters 21-59
      Called from Odoc_xref2__Subst.value in file "subst.ml" (inlined), line 842, characters 19-38
      Called from Odoc_xref2__Subst.apply_sig_map.inner.(fun) in file "subst.ml", line 1089, characters 19-52
      Called from Odoc_xref2__Component.Delayed.get in file "component.ml" (inlined), line 55, characters 16-22
      Called from Odoc_xref2__Lang_of.signature_items.inner in file "lang_of.ml", line 438, characters 16-39
      Called from Odoc_xref2__Lang_of.signature in file "lang_of.ml" (inlined), line 466, characters 12-43
      Called from Odoc_xref2__Lang_of.include_ in file "lang_of.ml", line 641, characters 18-69</pre>
<p>The key thing here is that definition of <code>'a t</code> in <code>S1</code> - a destructive substituion. If you type this code into an OCaml toplevel, you will see that the signature of <code>S1</code> is:</p>
<div><pre class="language-ocaml"><code>module type S1 = sig
  type t0
  type 'a t := unit

  val x : t0 t
end</code></pre></div>
<p>where the substitution has clearly taken place. In contrast, odoc takes the position that the use of these inline destructive substitutions is to make the code easier to understand, and so it tries to keep them in the signature rather than simply apply them and present the resulting signature. So when rendering <code>S1</code> we end up with:</p>

<div class="inset" style="border: 1px solid var(--pre-border-color); padding: 10px; border-radius: 5px">
<a class="anchor"></a><h2>Module type <code><span>S1</span></code></h2>
<div class="odoc-spec"><div class="spec type anchored"><a href="https://jon.recoil.org/atom.xml#type-t0" class="anchor"></a><code><span><span class="keyword">type</span> t0</span></code></div></div><div class="odoc-spec"><div class="spec type subst anchored"><a href="https://jon.recoil.org/atom.xml#type-t" class="anchor"></a><code><span><span class="keyword">type</span> <span>'a t</span></span><span> := unit</span></code></div></div><div class="odoc-spec"><div class="spec value anchored"><a href="https://jon.recoil.org/atom.xml#val-x" class="anchor"></a><code><span><span class="keyword">val</span> x : <span><a href="https://jon.recoil.org/atom.xml#type-t0">t0</a> <a href="https://jon.recoil.org/atom.xml#type-t">t</a></span></span></code></div></div>
</div>

<p>The reported problem is a failure with a stack trace while processing <code>S3</code>, but upon looking closely the real problem has happened when expanding <code>S2</code>. What happens is that we have a type <code>t</code> defined in <code>S2</code> and a type <code>t</code> that will later be substituted away that comes from the inclusion of <code>S1</code>. The rendered signature of <code>S2</code> is:</p>

<div class="inset" style="border: 1px solid var(--pre-border-color); padding: 10px; padding-right:30px; border-radius: 5px">
<a class="anchor"></a><h2>Module type <code><span>S2</span></code></h2>
<div class="odoc-spec"><div class="spec type anchored"><a href="https://jon.recoil.org/atom.xml#type-s2-t" class="anchor"></a><code><span><span class="keyword">type</span> t</span></code></div></div><div class="odoc-include"><details open="open"><summary class="spec include"><code><span><span class="keyword">include</span> <a href="https://jon.recoil.org/atom.xml#module-type-S1">S1</a> <span class="keyword">with</span> <span><span class="keyword">type</span> <a href="https://jon.recoil.org/atom.xml#type-t0">t0</a> := <a href="https://jon.recoil.org/atom.xml#type-s2-t">t</a></span></span></code></summary><div class="odoc-spec"><div class="spec type subst anchored"><a href="https://jon.recoil.org/atom.xml#type-s2-t" class="anchor"></a><code><span><span class="keyword">type</span> <span>'a t</span></span><span> := unit</span></code></div></div><div class="odoc-spec"><div class="spec value anchored"><a href="https://jon.recoil.org/atom.xml#val-x" class="anchor"></a><code><span><span class="keyword">val</span> x : <span><a href="https://jon.recoil.org/atom.xml#type-s2-t">t</a> <a href="https://jon.recoil.org/atom.xml#type-s2-t">t</a></span></span></code></div></div></details></div>
</div>

<p>where the type of <code>x</code> is now <code>t t</code>, which is clearly incorrect. The problem is that odoc assumes that type names are unique within a signature (modulo shadowing, which isn't quite what's going on here), but in this signature there are two definitions of <code>type t</code>, one of which is parameterised and one is not. At this point nothing fatal has happened, but when we try to process <code>S3</code> the substitution code gets very confused by these different arities and <code>List.fold_left2</code> throws the above exception.</p>
<p>The fix I'm trialling for this is that when we're including a signature that contains an inline destructive substitution, we will perform that substitution when the expansion of the include is done. This means that the rendered signature of <code>S1</code> will be just the same as before, but the rendered signature of <code>S2</code> will now be:</p>

<div class="inset" style="border: 1px solid var(--pre-border-color); padding: 10px; padding-right:30px; border-radius: 5px">
<a class="anchor"></a><h2>Module type <code><span>S2</span></code></h2>
<div class="odoc-spec"><div class="spec type anchored"><a href="https://jon.recoil.org/atom.xml#type-s2new-t" class="anchor"></a><code><span><span class="keyword">type</span> t</span></code></div></div><div class="odoc-include"><details open="open"><summary class="spec include"><code><span><span class="keyword">include</span> <a href="https://jon.recoil.org/atom.xml#module-type-S1">S1</a> <span class="keyword">with</span> <span><span class="keyword">type</span> <a href="https://jon.recoil.org/atom.xml#type-t0">t0</a> := <a href="https://jon.recoil.org/atom.xml#type-s2new-t">t</a></span></span></code></summary><div class="odoc-spec"><div class="spec value anchored"><a href="https://jon.recoil.org/atom.xml#val-x" class="anchor"></a><code><span><span class="keyword">val</span> x : unit</span></code></div></div></details></div>
</div>

<p>where the type of <code>x</code> is now simply <code>unit</code>, which is what OCaml itself thinks, happily! I think this strikes the balance between keeping the substitutions visible for clarity where they are originally defined, but when including them elsewhere we simply see the resulting signature.</p>
<h3><a href="https://jon.recoil.org/atom.xml#bug-%231385:-exception-raised-during-compilation" class="anchor"></a>Bug #1385: Exception raised during compilation</h3>
<p>The second bug has the identical backtrace, indicating a problem with arities. However, the repro case for this one does not involve any inline destructive substitution, though it does involve destructive substitution at the module expression level:</p>
<div><pre class="language-ocaml"><code>module type Creators_base = sig
  type ('a, _, _) t
  type (_, _, _) concat

  include sig
      type ('a, 'b, 'c) t

      val concat : (('a, 'p1, 'p2) t, 'p1, 'p2) concat -&gt; ('a, 'p1, 'p2) t
    end
    with type ('a, 'b, 'c) t := ('a, 'b, 'c) t
end

module type S0_with_creators_base = sig
  type t

  include Creators_base with type ('a, _, _) t := t and type ('a, _, _) concat := t
end</code></pre></div>
<p>There's quite a lot of type parameters flying around here, so the first step was to try to simplify this as much as possible while still getting the exception. I got it down to:</p>
<div><pre class="language-ocaml"><code>module type Creators_base = sig
  type 'a t
  type _ concat

  include sig
      type 'a t

      val concat : 'a concat -&gt; 'a t
    end
    with type 'a t := 'a t
end

module type S0_with_creators_base = sig
  type t

  include Creators_base with type _ t := t with type _ concat := t
end</code></pre></div>
<p>which still throws the same exception. So, what's going on here? Fundamentally, it's a similar issue to the first bug, just caused in a different way, in that once again we'll end up with a signature that has two definitions of <code>type t</code> with different arities. In this case, the problem occurs during the expansion of <code>S0_with_creators_base</code>.</p>
<p>This is the intermediate expansion of <code>Creators_base</code> that odoc calculates:</p>
<div><pre class="language-ocaml"><code>module type S0_with_creators_base = sig
  type t

  include Creators_base with type _ t := t with type _ concat := t (*
    
    The expansion as calculated by odoc is:

      include sig
        type 'a t
        val concat : t -&gt; 'a t
      end
  *)
end</code></pre></div>
<p>What's happened here is during the calculation of the body of the include, odoc has taken the signature of <code>Creators_base</code> and has its two type definitions both replaced with <code>type t</code> (with no parameters). However, since the <code>type t</code> in the body of the include is defined in that signature, that one wasn't replaced. So we end up with the type of <code>concat</code> being <code>t -&gt; 'a t</code>, which looks very odd! At this point though, odoc knows very well that they're different types. However, when odoc converts this signature back into the datatype that represents the expansions, it loses that information and we end up with the two types mixed up. We then go on to process this signature, the mixup of the arities causes the failure.</p>
<p>There are several independent fixes that we can make here. Firstly we can make sure that we don't mix up the types. This we can do because we can distinguish between items that are declared within the signature of the include's declaration and those that come from the outer context. We don't have to do this for the expansion of the include as OCaml's type system means that there can't be two types of the same name in the resulting signature. We never actually render any signature that occurs within the body of an include, so this doesn't actually make any difference to the output.</p>
<p>The second fix is to make sure that we only calculate the expansion of the include once. Currently the bug happens because we try to re-calculate the expansion of the <code>include sig ... end</code> expression, even though we calculated it during the processing of <code>S0_with_creators_base</code>. What we should do instead is apply the substitutions to the expansion of that calculated include, which would end up with the same result. This isn't a perfect solution though, as there are occasions when we have to recalculate the signature anyway.</p>
<p>The third fix is - and this takes a little care to parse - to ensure that we never actually try to process the items within a signature within a "with" expression within a module-type expression. Before diving into the 'why' of this, let's first explain how Odoc represents module-type expressions.</p>
<p>Internally, we have <a href="https://jon.recoil.org/reference/odoc/odoc.model/Odoc_model/Lang/ModuleType/index.html#type-expr" title="Odoc_model.Lang.ModuleType.expr">a datatype</a> that represents module expressions, which looks like this:</p>
<div><pre class="language-ocaml"><code>type expr =
    | Path of path_t
    | Signature of Signature.t
    | Functor of FunctorParameter.t * expr
    | With of with_t
    | TypeOf of typeof_t</code></pre></div>
<p>Now, each of the arguments to these constructors might contain an expansion of the expression that Odoc will calculate. For example, the definition of <a href="https://jon.recoil.org/reference/odoc/odoc.model/Odoc_model/Lang/ModuleType/index.html#type-path_t" title="Odoc_model.Lang.ModuleType.path_t">path_t</a> is:</p>
<div><pre class="language-ocaml"><code>type path_t = {
    p_expansion : simple_expansion option;
    p_path : Paths.Path.ModuleType.t;
}</code></pre></div>
<p>and this expansion is initially <code>None</code> and then filled in by Odoc in order to render the expansion in the HTML. In the case of a <code>With</code> expression, the <a href="https://jon.recoil.org/reference/odoc/odoc.model/Odoc_model/Lang/ModuleType/index.html#type-with_t" title="Odoc_model.Lang.ModuleType.with_t">with_t</a> type is:</p>
<div><pre class="language-ocaml"><code>type with_t = {
    w_substitutions : substitution list;
    w_expansion : simple_expansion option;
    w_expr : U.expr;
}</code></pre></div>
<p>here you can see that the <code>With</code> expression contains another module expression, as a <code>with</code> expression operates on another module type. Early during Odoc's development, this simply was another `ModuleType.expr`, but we had a couple of bugs where we ended up calculating expansions for these inner expressions, which was all very wasteful as we only ever rendered the "outer" expansion. So we changed this to be a <a href="https://jon.recoil.org/reference/odoc/odoc.model/Odoc_model/Lang/ModuleType/U/index.html#type-expr" title="Odoc_model.Lang.ModuleType.U.expr">U.expr</a>, which is an "unexpanded" module type expression, and is very similar to the main expression above, but without the expansions and also with the functor case, as we can't have functors inside a "with" expression.</p>
<p>These "unexpanded" expressions still contain signatures though, so aren't <em>completely</em> unexpanded, and it's <em>these</em> signatures that we should avoid processing.</p>
<p>So, what I expected to be just one bug when I started looking at this turned out to be two related issues, and a total of four different fixes!</p>
