The point of the `pandoc-filter` crate is to make it easier to write
transformations on the pandoc abstract syntax tree (AST), so that you
can manipulate the structure of a given markdown document before
finally converting it to the final output product.

To do this, we need to interact with the AST, which pandoc represents
as a JSON object. I used to resort to working with the JSON via its
represntation as defined in the `rustc_serialize` crate, but a better
option is the `pandoc_ast` crate.

```rust
extern crate pandoc_ast;

use pandoc_ast::{Pandoc, Map, Meta, MetaValue, Block, Inline, Attr, Target, Citation};

use pandoc_ast::{Format, QuoteType, TableCell, Int, ListNumberStyle, ListNumberDelim};
use pandoc_ast::{Alignment, Double, CitationMode};

pub trait PandocVisitor: Sized {
    fn visit_pandoc(&mut self, p: &mut Pandoc) { walk_pandoc(self, p); }
    fn visit_meta(&mut self, m: &mut Meta) { walk_meta(self, m); }
    fn visit_meta_entry(&mut self, k: &str, v: &mut MetaValue) { walk_meta_entry(self, k, v); }
    fn visit_meta_value(&mut self, v: &mut MetaValue) { walk_meta_value(self, v); }
    fn visit_inline(&mut self, i: &mut Inline) { walk_inline(self, i); }
    fn visit_block(&mut self, b: &mut Block) { walk_block(self, b); }
    fn visit_attr(&mut self, a: &mut Attr) { walk_attr(self, a); }
    fn visit_target(&mut self, t: &mut Target) { walk_target(self, t); }
    fn visit_citation(&mut self, c: &mut Citation) { walk_citation(self, c); }
    fn visit_format(&mut self, f: &mut Format) { walk_format(self, f); }
    fn visit_table_cell(&mut self, c: &mut TableCell) { blocks(self, c); }
}

pub fn walk_pandoc<V>(s: &mut V, p: &mut Pandoc) where V: PandocVisitor {
    let Pandoc(ref mut meta, ref mut blocks) = *p;
    s.visit_meta(meta);
    self::blocks(s, blocks);
}

pub fn walk_meta<V>(s: &mut V, m: &mut Meta) where V: PandocVisitor {
    meta_map(s, &mut m.unMeta);
}

fn meta_map<V>(s: &mut V, m: &mut Map<String, MetaValue>) where V: PandocVisitor {
    for (k, v) in m { s.visit_meta_entry(k, v); }
}

fn meta_map_boxed<V>(s: &mut V, m: &mut Map<String, Box<MetaValue>>) where V: PandocVisitor {
    for (k, v) in m { s.visit_meta_entry(k, v); }
}

fn meta_list<V>(s: &mut V, m: &mut Vec<MetaValue>) where V: PandocVisitor {
    for v in m { s.visit_meta_value(v); }
}

fn inlines<V>(s: &mut V, v: &mut Vec<Inline>) where V: PandocVisitor {
    for i in v { s.visit_inline(i); }
}

fn blocks<V>(s: &mut V, v: &mut Vec<Block>) where V: PandocVisitor {
    for b in v { s.visit_block(b); }
}

fn citations<V>(s: &mut V, v: &mut Vec<Citation>) where V: PandocVisitor {
    for c in v { s.visit_citation(c); }
}

pub fn walk_meta_entry<V>(s: &mut V, _: &str, v: &mut MetaValue) where V: PandocVisitor {
    s.visit_meta_value(v);
}

pub fn walk_meta_value<V>(s: &mut V, v: &mut MetaValue) where V: PandocVisitor {
    match *v {
        MetaValue::MetaMap(ref mut m) => meta_map_boxed(s, m),
        MetaValue::MetaList(ref mut v) => meta_list(s, v),
        MetaValue::MetaBool(_) |
        MetaValue::MetaString(_) => {}
        MetaValue::MetaInlines(ref mut v) => inlines(s, v),
        MetaValue::MetaBlocks(ref mut v) => blocks(s, v),
    }
}

pub fn walk_inline<V>(s: &mut V, i: &mut Inline) where V: PandocVisitor {
    match *i {
        Inline::Space |
        Inline::SoftBreak |
        Inline::LineBreak => {}

        Inline::Str(ref content) => { let _: &String = content; }

        Inline::Emph(ref mut inlines) |
        Inline::Strong(ref mut inlines) |
        Inline::Strikeout(ref mut inlines) |
        Inline::Superscript(ref mut inlines) |
        Inline::Subscript(ref mut inlines) |
        Inline::SmallCaps(ref mut inlines) => self::inlines(s, inlines),

        Inline::Quoted(ref _type, ref mut inlines) => {
            let _: &QuoteType = _type;
            self::inlines(s, inlines);
        }

        Inline::Cite(ref mut citations, ref mut inlines) => {
            self::citations(s, citations);
            self::inlines(s, inlines);
        }

        Inline::Code(ref mut attr, ref code) => {
            s.visit_attr(attr);
            let _: &String = code;
        }
        Inline::RawInline(ref mut format, ref mut content) => {
            s.visit_format(format);
            let _: &String = content;
        }
        Inline::Math(_, ref content) => {
            let _: &String = content;
        }
        Inline::Link(ref mut attr, ref mut inlines, ref mut target) |
        Inline::Image(ref mut attr, ref mut inlines, ref mut target) => {
            s.visit_attr(attr);
            self::inlines(s, inlines);
            s.visit_target(target);
        }
        Inline::Note(ref mut blocks) => self::blocks(s, blocks),
        Inline::Span(ref mut attr, ref mut inlines) => {
            s.visit_attr(attr);
            self::inlines(s, inlines);
        }
    }
}

pub fn walk_attr<V>(_s: &mut V, a: &mut Attr) where V: PandocVisitor {
    let _: &mut (String, Vec<String>, Vec<(String, String)>) = a;
}
pub fn walk_target<V>(_s: &mut V, t: &mut Target) where V: PandocVisitor {
    let _: &mut (String, String) = t;
}

pub fn walk_block<V>(s: &mut V, b: &mut Block) where V: PandocVisitor {
    match *b {
        Block::Plain(ref mut inlines) |
        Block::Para(ref mut inlines) => self::inlines(s, inlines),
        Block::CodeBlock(ref mut attr, ref mut content) => {
            s.visit_attr(attr);
            let _: &mut String = content;
        }
        Block::RawBlock(ref mut format, ref mut content) => {
            s.visit_format(format);
            let _: &mut String = content;
        }
        Block::BlockQuote(ref mut blocks) => self::blocks(s, blocks),
        Block::OrderedList(ref mut list_attrs, ref mut vec_blocks) => {
            let _: &mut (Int, ListNumberStyle, ListNumberDelim) = list_attrs;
            for blocks in vec_blocks { self::blocks(s, blocks); }
        }
        Block::BulletList(ref mut vec_blocks) => {
            for blocks in vec_blocks { self::blocks(s, blocks); }
        }
        Block::DefinitionList(ref mut defns) => {
            for &mut (ref mut inlines, ref mut vec_blocks) in defns {
                self::inlines(s, inlines);
                for blocks in vec_blocks { self::blocks(s, blocks); }
            }
        }
        Block::Header(ref i, ref mut attr, ref mut inlines) => {
            let _: &Int = i;
            s.visit_attr(attr);
            self::inlines(s, inlines);
        }
        Block::HorizontalRule => {}
        Block::Table(ref mut inlines, ref aligns, ref widths,
                     ref mut headers, ref mut rows) => {
            self::inlines(s, inlines);
            let _: &Vec<Alignment> = aligns;
            let _: &Vec<Double> = widths;
            for cell in headers { s.visit_table_cell(cell); }
            for row in rows {
                for cell in row { s.visit_table_cell(cell); }
            }
        }
        Block::Div(ref mut attr, ref mut blocks) => {
            s.visit_attr(attr);
            self::blocks(s, blocks);
        }
        Block::Null => {}
    }
}

pub fn walk_citation<V>(s: &mut V, c: &mut Citation) where V: PandocVisitor {
    #![allow(non_snake_case)]
    let Citation { ref citationId, ref mut citationPrefix, ref mut citationSuffix,
                   ref citationMode, ref citationNoteNum, ref citationHash } = *c;
    let _: &String = citationId;
    self::inlines(s, citationPrefix);
    self::inlines(s, citationSuffix);
    let _: &CitationMode = citationMode;
    let _: &Int = citationNoteNum;
    let _: &Int = citationHash;
}

pub fn walk_format<V>(_s: &mut V, f: &mut Format) where V: PandocVisitor {
    let Format(ref s) = *f;
    let _: &String = s;
}
```
