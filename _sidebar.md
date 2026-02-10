<!-- Sidebar Navigation -->

* [🏠 Home](/)
* [📊 Database Schema](DATABASE-SCHEMA.md)

* **Component Design**
  * **Stores**
    * [Book Store](component/stores/book-store.md)
    * [Snapshot Store](component/stores/snapshot-store.md)
    * [Editor Settings Store](component/stores/editor-settings-store.md)
  * [Editor Page](component/editor-page/00-editor-page.md)
    * [Editor Header](component/editor-page/01-editor-header.md)
    * [Icon Rail](component/editor-page/02-icon-rail.md)
    * [Doc CreativeSpace](component/editor-page/doc-creative-space/00-doc-creative-space.md)
      * [Doc Sidebar](component/editor-page/doc-creative-space/01-doc-sidebar.md)
      * [Doc Editor](component/editor-page/doc-creative-space/02-manuscript-doc-editor.md)
    * [Dummy CreativeSpace](component/editor-page/dummy-creative-space/00-dummy-creative-space.md)
      * [Dummy Sidebar](component/editor-page/dummy-creative-space/01-dummy-sidebar.md)
    * [Sketch CreativeSpace](component/editor-page/sketch-creative-space/00-sketch-creative-space.md)
      * [Sketch Sidebar](component/editor-page/sketch-creative-space/01-sketch-sidebar.md)
      * [Sketch Viewer](component/editor-page/sketch-creative-space/02-sketch-viewer.md)
  * **Shared Components**
    * [Manuscript Spread View](component/editor-page/shared/manuscript-spread-view/00-manuscript-spread-view.md)
      * [Spread View Header](component/editor-page/shared/manuscript-spread-view/01-spread-view-header.md)
      * [Spread Editor Panel](component/editor-page/shared/manuscript-spread-view/02-spread-editor-panel.md)
        * [Editable Image](component/editor-page/shared/manuscript-spread-view/02-01-editable-image.md)
        * [Editable Textbox](component/editor-page/shared/manuscript-spread-view/02-02-editable-textbox.md)
        * [Selection Frame](component/editor-page/shared/manuscript-spread-view/02-03-selection-frame.md)
      * [Spread Thumbnail List](component/editor-page/shared/manuscript-spread-view/03-spread-thumbnail-list.md)
        * [Spread Thumbnail](component/editor-page/shared/manuscript-spread-view/03-01-spread-thumbnail.md)

* **App Features**
  * [Story Idea Brainstorming](app/ai-assistant/story-idea-brainstorming.md)
  * [Generate Manuscript](app/generate-manuscript/generate-manuscript.md)

* **API - Chat**
  * [00 - Brainstorming Initial](api/chat/00-story-brainstorming-initial.md)
  * [01 - Brainstorming](api/chat/01-story-brainstorming.md)

* **API - Text Generation**
  * [00 - Generate Manuscript](api/text-generation/00-generate-manuscript.md)
  * [01 - Story Draft](api/text-generation/01-generate-story-draft.md)
  * [02 - Spread Visual Plan](api/text-generation/02-generate-spread-visual-plan.md)
  * [03 - Text Refinement](api/text-generation/03-generate-text-refinement.md)
  * [04 - Spread Composition](api/text-generation/04-generate-spread-composition.md)
  * [05 - Quality Check](api/text-generation/05-generate-quality-check.md)

* **API - Visual Description**
  * [06 - Character](api/text-generation/06-generate-visual-description-character.md)
  * [07 - Prop](api/text-generation/07-generate-visual-description-prop.md)
  * [08 - Stage](api/text-generation/08-generate-visual-description-stage.md)
  * [09 - Spread](api/text-generation/09-generate-visual-description-spread.md)

* **API - Translation & Poetry**
  * [10 - Translate Content](api/text-generation/10-translate-content.md)
  * [11 - Generate Poetry](api/text-generation/11-generate-poetry.md)
