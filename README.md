# Guide: Writing Robust GIMP 3.0 Python Plugins on macOS (MacBook Pro M1, MacPorts GIMP)

This document provides a technical guide for programmers wishing to write functional GIMP 3.0 plugins in Python. It focuses on the constraints and solutions discovered while working with a macOS system (Apple Silicon M1) using a MacPorts-installed version of GIMP 3.0. This guide is based on direct empirical testing and aims to highlight working patterns and help avoid common pitfalls.

## ❗ Critical Setup & Environment Requirements

**Strict adherence to these points is essential for your plug-in to be recognized by GIMP 3.0 on this platform.**

* **System:** macOS (Apple Silicon, M1 architecture confirmed)
* **GIMP Version:** GIMP 3.0.x (specifically tested with a MacPorts installation)
* **Python Interface:** **GObject Introspection (GI) only.** Do not attempt to use the GIMP 2.x `gimpfu` module; it will cause a `ModuleNotFoundError`.
    ```python
    import gi
    gi.require_version('Gimp', '3.0')
    gi.require_version('GimpUi', '3.0') # If using GimpUi elements
    from gi.repository import Gimp, GimpUi, GObject, GLib
    import sys
    ```
* **Plug-in File Location & Structure (Absolutely Critical):**
    1.  **User Plug-in Directory:** `~/Library/Application Support/GIMP/3.0/plug-ins/`
    2.  **Subdirectory Requirement:** Your Python script (e.g., `my_plugin_script.py`) **MUST** be placed inside its own dedicated subdirectory within the `plug-ins/` folder. GIMP will skip scripts placed directly in the `plug-ins/` root.
    3.  **Subdirectory Naming (Empirically Critical):** For reliable loading in the tested MacPorts environment, the subdirectory name **MUST exactly match the Python script's filename (without the `.py` extension).**
        * Correct: `~/Library/Application Support/GIMP/3.0/plug-ins/my_plugin_script/my_plugin_script.py`
        * Incorrect (and will likely fail to load): `~/Library/Application Support/GIMP/3.0/plug-ins/some_other_folder_name/my_plugin_script.py`
        * Incorrect (and will fail to load): `~/Library/Application Support/GIMP/3.0/plug-ins/my_plugin_script.py`
* **Execution Environment:** The Python script file **MUST** be executable.
    ```bash
    chmod +x ~/Library/Application\ Support/GIMP/3.0/plug-ins/your_script_folder/your_script_file.py
    ```
* **Shebang Line:** The script must start with the correct shebang line:
    ```python
    #!/usr/bin/env python3
    # -*- coding: utf-8 -*- # Good practice for encoding
    ```

## 🛠️ Essential Debugging Techniques for macOS (MacPorts GIMP 3.0)

These were indispensable for diagnosing issues:

1.  **Launch GIMP from Terminal (Primary Debugging Tool):**
    * Always launch GIMP from the Terminal when developing or troubleshooting plug-ins:
        ```bash
        /Applications/GIMP.app/Contents/MacOS/gimp 
        # Or use an alias if you've set one up:
        # gimp 
        ```
    * The terminal will display Python `AttributeError`s, tracebacks, GIMP warnings, and other messages that are often *not visible* in GIMP's GUI Error Console. This is the most reliable way to see why a plug-in might be failing to load or run.

2.  **Use the Procedure Browser (Not the Plug-in Browser):**
    * **Procedure Browser (Help > Procedure Browser):** This is the definitive tool to check if your plug-in's procedures have been successfully registered with GIMP's Procedural Database (PDB). Search for the PDB name you defined in `do_query_procedures()` (e.g., `"plugin-path-crop"`).
    * **Plug-in Browser (Help > Plug-in Browser):** In the tested MacPorts GIMP 3.0.2 environment, this dialog was found to be unreliable or non-functional (e.g., showing "0 plug-ins" or producing its own errors in the terminal). **Do not rely on it** to verify if your plug-in is loaded.

3.  **Iterative PDB Call Strategy:**
    When a GIMP function is needed:
    a.  **Attempt Direct GObject Method:** First, try to find and use a direct method on the relevant GIMP object (e.g., `image.crop(...)`, `drawable.edit_clear()`). This is the preferred GIMP 3 style.
    b.  **If Direct Method Fails (AttributeError):** Resort to the robust `Gimp.get_pdb().lookup_procedure("gimp-pdb-function-name")` mechanism. Create a config, set properties, and run.
    c.  **Avoid Direct `pdb.gimp_function_name(...)`:** Most of these GIMP 2.x style direct PDB attribute calls (e.g., `pdb.gimp_image_get_active_vectors()`) were found to be unavailable (`AttributeError`) on the object returned by `Gimp.get_pdb()` in the tested environment.

4.  **Logging:**
    * Use `Gimp.message(f"Your debug message: {variable}")` to print messages to GIMP's Error Console (and often the terminal).
    * Use `print(f"Terminal debug: {variable}")` for messages that should *only* go to the terminal from which GIMP was launched.

## ✅ Core Plug-in Structure & API Usage (GIMP 3.0 GI)

1.  **Subclass `Gimp.PlugIn`:**
    ```python
    class MyPluginClassName(Gimp.PlugIn):
        # ... plugin methods ...
    ```

2.  **Implement `do_query_procedures(self)`:**
    * Return a list containing the unique PDB name(s) for your plug-in's procedure(s).
    ```python
    def do_query_procedures(self):
        return ["com-yourdomain-my-plugin-action"] # Use a unique, namespaced PDB name
    ```

3.  **Implement `do_create_procedure(self, name)`:**
    * Create and configure a `Gimp.Procedure` (usually `Gimp.ImageProcedure` if it operates on an image).
    ```python
    def do_create_procedure(self, name):
        procedure = Gimp.ImageProcedure.new(self, name,
                                            Gimp.PDBProcType.PLUGIN,
                                            self.run, # Your main logic function
                                            None)   # No custom Gimp.Config object
        # Set essential properties:
        procedure.set_image_types("*")
        procedure.set_sensitivity_mask(Gimp.ProcedureSensitivityMask.DRAWABLE) # Or other relevant masks
        procedure.set_menu_label(_("My Plugin Menu Label"))
        procedure.add_menu_path("<Image>/Filters/MyCustomMenu/")
        procedure.set_documentation(
            _("Short help string for Procedure Browser & tooltips."), # This is often used as the blurb
            _("More detailed help text for the plug-in (longer description)."),
            name # Or a custom help ID for context help
        )
        procedure.set_attribution("Author Name", "Copyright Holder", "Year")
        # procedure.set_blurb("...") was found to be unavailable (AttributeError) in tested GIMP 3.0.2 MacPorts.
        # The first argument to set_documentation() often serves as the blurb.
        return procedure
    ```

4.  **Implement `do_set_i18n(self, procedure_name)` (Simplified):**
    * Due to `AttributeError`s with PDB/GLib localization functions in the tested environment, a simplified version is recommended to prevent load failures:
    ```python
    def do_set_i18n(self, procedure_name):
        # global TEXT_DOMAIN # If you have a global for your _() function
        # TEXT_DOMAIN = procedure_name
        return # Or return False; effectively disables GIMP's advanced i18n for this plug-in
    ```
    * Define a simple `_()` for untranslated strings if needed: `def _(message): return message`

5.  **Implement the `run(...)` Method:**
    * Signature: `def run(self, procedure, run_mode, image, drawables, config, run_data):`
    * Get the primary drawable: `drawable = drawables[0]` (after checking `drawables` is not empty).
    * Use GObject methods and `lookup_procedure` as determined by testing.

6.  **Register with `Gimp.main()`:**
    ```python
    Gimp.main(MyPluginClassName.__gtype__, sys.argv)
    ```

## ✅ Confirmed Working API Calls & Patterns (from our debugging)

* **Plug-in Structure:** As outlined above.
* **Path Retrieval:** `paths = image.get_paths()` then `active_path = paths[0]` (if `paths` is not empty).
* **Path to Selection:** `image.select_item(Gimp.ChannelOps.REPLACE, active_path_object)`
* **Selection Operations:** `Gimp.Selection.invert(image)`
* **Drawable Operations:** `drawable.edit_clear()`, `drawable.get_name()`, `drawable.get_id()`
* **Image Operations:** `image.undo_group_start()`, `image.undo_group_end()`, `image.crop(w, h, x, y)`, `image.get_id()`
* **PDB Lookup (for calls where direct methods failed):**
    * `gimp-selection-bounds`:
        ```python
        bounds_proc = Gimp.get_pdb().lookup_procedure("gimp-selection-bounds")
        cfg = bounds_proc.create_config()
        cfg.set_property("image", image)
        result = bounds_proc.run(cfg)
        # Unpack: status=result.index(0), exists=result.index(1), x1=result.index(2), ...
        ```
* **Display Update:** `Gimp.displays_flush()`
* **Error Handling:** `try...except Exception as e:`, returning `procedure.new_return_values(Gimp.PDBStatusType.CALLING_ERROR / EXECUTION_ERROR, GLib.Error.new_literal(Gimp.PlugIn.error_quark(), msg, 0))`
* **Success Return:** `procedure.new_return_values(Gimp.PDBStatusType.SUCCESS, None)`
* **Logging:** `Gimp.message("...")`

## ❌ Non-Functional or Problematic API Calls (in tested MacPorts GIMP 3.0.2)

This list reflects what **failed or was unreliable during our specific debugging session**. Some may work in other GIMP 3 builds or future versions.

* **`gimpfu` module:** Completely unsupported (causes `ModuleNotFoundError`).
* **Direct `pdb.gimp_function_name(...)` calls:** Most failed with `AttributeError` on the `pdb` object (e.g., `pdb.gimp_image_get_active_vectors`, `pdb.gimp_vectors_to_selection`, `pdb.gimp_plug_in_set_translation_domain`). **Use `lookup_procedure()` instead.**
* **`image.get_active_vectors()`:** `AttributeError`.
* **`image.get_selected_paths()`:** While documented for GIMP 3.0, `image.get_paths()` proved more reliable in testing. This might be specific to the build.
* **`image.selection_bounds()` (or `image.get_selection_bounds()`):** `AttributeError`. Must use `lookup_procedure("gimp-selection-bounds")`.
* **`image.undo_is_active()`:** `AttributeError`. Avoid; simply pair `undo_group_start()` and `undo_group_end()` carefully (e.g., with a flag in `try...finally` or `try...except...finally`).
* **`GLib.bindtextdomain(...)`:** `AttributeError` on the `GLib` object.
* **`Gimp.ProcedureSensitivityMask.VECTORS` (and `.PATHS`):** Caused `AttributeError`. Use `Gimp.ProcedureSensitivityMask.DRAWABLE` and perform explicit checks for paths in the `run` method.
* **`image.get_active_layer()`:** Not the GIMP 3 way. Use `drawables[0]` from the `run` arguments, or `image.get_active_drawable()`, or `image.get_selected_layers()`.
* **`procedure.set_blurb("...")`:** `AttributeError: 'ImageProcedure' object has no attribute 'set_blurb'`. The first argument to `procedure.set_documentation()` often serves as the blurb/short help.

## ⚠️ Important Considerations

* **Error Handling:** Robust `try...except` blocks are essential. Ensure `image.undo_group_end()` is called if `image.undo_group_start()` succeeded, even if an error occurs.
* **PDB Procedure Names:** When using `lookup_procedure()`, ensure you have the correct PDB string name (e.g., `"gimp-selection-bounds"`, not `"gimp_selection_bounds"`).
* **Return Values from `lookup_procedure().run()`:** These are `Gimp.ValueArray` objects. The first element (`result.index(0)`) is always the `Gimp.PDBStatusType`. Subsequent elements (`result.index(1)`, `result.index(2)`, etc.) are the actual return values of the PDB procedure.

## Conclusion

Writing Python plug-ins for GIMP 3.0, especially in specific environments like MacPorts on macOS, requires careful attention to the GObject Introspection API and empirical testing. By following the patterns confirmed to work (direct object methods where available, robust PDB lookups otherwise) and avoiding known pitfalls, you can create functional and stable plug-ins. Always prioritize clear error handling and use terminal-based launching for effective debugging.
