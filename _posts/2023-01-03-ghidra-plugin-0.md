# Making a Ghidra Plugin, Part 0/N: Hello, PCode
I've done a ton of automation with Ghidra, but it's been purely in the form of Python (Jython) and Java scripting, driven by Ghidra's `analyzeHeadless` capability. I've never extended the GUI in any way, and doing so has always looked like Black Magic to me. What better way to fix that than by authoring a blog series on it?

[CAVEAT] I am not an expert Java developer. I hobble along, assisted with an IDE (Eclipse) and a plugin (GhidraDev) that does most of the work for me. I will most certainly do things that are curious to a seasoned Java developer. Please feel free to help me learn gud in the form of issues and PRs!

## Setting the Table
The goal for this post is to develop the motivation for this tool, make a plan, and execute step 1 (a basic functioning GUI plugin). Future posts will add features as laid out in the plan, and along the way I hope to document the process and workflow of developing a Ghidra plugin.

## Step 0: Identify the Need
Before I start willy-nilly diving into GhidraDev-land, let's talk about the problem I want to solve. I was recently developing some tooling to analyze Ghidra's PCode, which is the intermediate language (IL) used by Ghidra to improve disassembly and implement the decompiler functionality. Analyzing the intermediate representation is incredibly powerful because you can create analysis primitives that span all architectures supported by that IL and avoid making tools that only operate on disassembly, which you need to recreate for every architecture that you want to support.

While you can enable a PCode display in the Listing view, it's not the full picture. PCode goes through several enhancement passes on the way to emitting a decompiled approximation of a function (analogous to Binary Ninja's levels of IL), but viewing the different results of these enhancements can be a little tricky to view and track down. The Ghidra decompiler docs are great for learning about how this is done. Note: you must build them from source using `make doc` in `ghidra/Ghidra/Features/Decompiler/src/decompile/cpp` (requires Doxygen).

As I dug deeper into PCode, it wasn't clear to me what impacts the different passes had, and while I was very familiar with Binary Ninja's BNIL, LLVM IR, and VEX IR, I had a difficult time drawing parallels between PCode and other static single assignment (SSA) ILs. Later passes of PCode transformations _do_ become SSA, but when printing the PCode operations for this code, the SSA-ness of it isn't immediately apparent. I made a Python script to dump PCode for a specified "style" ("style" is the Ghidra parlance for an enhancement pass result), as well as a "decorator" pass that added versioning information to the varnodes when displaying them, making it much easier to visually track dataflow across enhanced PCode. This was the "aha" moment that really unlocked PCode analysis for me, and I went on about the larger task at hand. However, I always wanted to turn this into a view in the GUI similar to the Listing and Decompile views.

## Step 1: Have a Plan
### Requirements
1. A GUI window that displays a configurable style of PCode for the function represented by the `currentLocation` in the `currentProgram`
2. A capability to display the PCode as a graph, like the Function Graph view
3. A button to toggle varnode versions on/off
4. The displayed PCode should be interactive (e.g., navigable like other views)

### Rough Plan
I'm going to attempt to follow this plan while developing (and learning) about how Ghidra GUI APIs and services work, and each one of these steps will be a post.
1. Get sample GUI plugin working, render raw (lowest level PCode) in a new window
2. Implement graph view capability
3. Add rendering for remaining higher levels of PCode
4. Add varnode decoration
5. Add interactivity

## Step 2: Getting Started
Get your development environment up and running. This consists of installing Eclipse, and the Eclipse GhidraDev plugin that comes with your installed version of Ghidra. Instructions for accomplishing this are in `<GhidraInstallDir>/Extensions/Eclipse/GhidraDev/GhidraDev_README.html`.

I started with the [ShowInfoPlugin](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Extensions/sample/src/main/java/ghidra/examples/ShowInfoPlugin.java) example, which is located under `Ghidra/Extensions/sample` in the Ghidra source tree. This folder contains a bunch of different great examples, and if you need something more complex, a great deal of Ghidra functionality is actually implemented as plugins, so you can just locate the code responsible for a feature you're interested in using, cross-reference it, and find examples in the Ghidra source. 

I created a new GhidraDev Ghidra Module project, and for the module template, I selected `Plugin`, leaving all others unchecked, since I'm just extending the GUI. Following the wizard, I linked my local Ghidra installation, and deselected Python support. Finishing the wizard leaves you with a skeleton project with a bunch of auto-generated folders and files, but the main one we're interested in is under `src/main/java`: `PCodeViewerPlugin.java`.

### Key Components
The primary things needed for a GUI plugin are the `ProgramPlugin` class, which manages the integration of plugin functionality and the Ghidra event system, and the `ComponentProvider` class, which is used to implement the functionality. The template class generated by the GhidraDev wizard dumps this all into the PCodeViewerPlugin.java file, but the first thing I did was follow the suggestion in the code and move the provider class to its own file. Then I copied the main elements from the ShowInfoPlugin code into my plugin, built it, installed it using Ghidra's File &rarr; Install Extensions... menu. I pointed this at the zip file that was built in the project's `dist` folder, restarted Ghidra, and ran the plugin via Window &rarr; PCodeViewerPlugin to ensure it was working properly. If you don't see the menu item, click the little computer at the bottom right of your Ghidra's project window and look for error messages indicating why the plugin didn't load.

#### ProgramPlugin
The main job of the `ProgramPlugin` for now is to initialize our provider class and handle events that we care about from the core. I want to render the provider every time a new function is selected, so the `locationChanged` event is what I want to register for. Resources should be cleaned up whenever the program is closed, so `programDeactivated` is another event to handle.

```Java
/**
 * This plugin provides a view into the current function's PCode.
 */
//@formatter:off
@PluginInfo(
	status = PluginStatus.STABLE,
	packageName = PCodeViewerPlugin.PLUGIN_NAME,
	category = PluginCategoryNames.CODE_VIEWER,
	shortDescription = "Explore different levels of PCode for a function.",
	description = "PCode view creates a PCode-specific view to separately "
			+ "render the different levels of PCode for a function."
)
//@formatter:on
public class PCodeViewerPlugin extends ProgramPlugin {
	public static final String PLUGIN_NAME = "PCodeViewer";

	PCodeProvider provider;

	/**
	 * PCodeViewerPlugin constructor.
	 * 
	 * @param tool The plugin tool that this plugin is added to.
	 */
	public PCodeViewerPlugin(PluginTool tool) {
		super(tool);

		String pluginName = getName();
		provider = new PCodeProvider(this, pluginName);

		// TODO: Customize help (or remove if help is not desired)
		String topicName = this.getClass().getPackage().getName();
		String anchorName = "HelpAnchor";
		provider.setHelpLocation(new HelpLocation(topicName, anchorName));
	}

	@Override
	public void init() {
		super.init();

		// TODO: Acquire services if necessary
	}
	
	@Override
	protected void programDeactivated(Program program) {
		provider.clear();
	}
	
	@Override
	protected void locationChanged(ProgramLocation loc) {
		provider.locationChanged(currentProgram, loc);
	}

}
```

#### ComponentProvider
The `ComponentProvider` implementation is responsible for _providing_ (heh) the bulk of the functionality we need. It handles building the components of the GUI window, and provides the methods to update the view when invoked by the plugin event handler.
```Java
public class PCodeProvider extends ComponentProviderAdapter {

	private JPanel panel;
	private JTextArea textArea;
	private DockingAction action;
	private Program currentProgram;
	private ProgramLocation currentLocation;
	private PCodeViewerPlugin plugin;

	public PCodeProvider(Plugin plugin, String owner) {
		super(plugin.getTool(), owner, owner);
		this.plugin = (PCodeViewerPlugin) plugin;
		buildPanel();
		createActions();
	}

	// Customize GUI
	private void buildPanel() {
		panel = new JPanel(new BorderLayout());
		textArea = new JTextArea(40, 80);
		textArea.setEditable(false);
		textArea.append(plugin.getName() + ": Set the current location inside a function to view its PCode");
		panel.add(new JScrollPane(textArea));
		setVisible(true);
	}

	// TODO: Customize actions
	private void createActions() {
		action = new DockingAction("My Action", getName()) {
			@Override
			public void actionPerformed(ActionContext context) {
				Msg.showInfo(getClass(), panel, "Custom Action", "Hello!");
			}
		};
		action.setToolBarData(new ToolBarData(Icons.ADD_ICON, null));
		action.setEnabled(true);
		action.markHelpUnnecessary();
		dockingTool.addLocalAction(this, action);
	}

	@Override
	public JComponent getComponent() {
		return panel;
	}
	
	public void locationChanged(Program program, ProgramLocation location) {
		currentProgram = program;
		currentLocation = location;
		if (isVisible()) {
			updatePanel();
		}
	}
	
	private String getPcode(Function f) {
		String pcodeString = PCodeUtils.rawPCodeString(currentProgram, f);
		return pcodeString;
	}
	
	private void updatePanel() {
		// Update the panel information, but only if the function scope has changed.
		Function func = currentProgram.getListing().getFunctionContaining(currentLocation.getAddress());
		if (func != null) {
			textArea.setText(getPcode(func));
		}
		else {
			textArea.setText("Address " + currentLocation.getAddress().toString() + " is not contained in a function.");
		}
	}

	public void clear() {
		// Clear stuff as needed
		currentProgram = null;
		currentLocation = null;
		textArea.setText("");
	}
}
```
### Displaying PCode
Now that we have our main plugin components up and running, we need to make them actually do something useful beyond render the UI containers. This is a pretty crude approach for the moment, but it gets me to the end of Step 1 in my plan, so I can claim success and get started on Step 2. I'll probably change this to use a basic block-based API when I implement the graph rendering.

```Java
public final class PCodeUtils {
	private PCodeUtils() {
		throw new UnsupportedOperationException();
	}
	
	public static String rawPCodeString(Program p, Function f) {
		// Dump the function's raw PCode as a string prefixed by the address.
		String pcodeStr = new String();
		AddressSetView body = f.getBody();
		InstructionIterator instrIter = p.getListing().getInstructions(body, true);
		while (instrIter.hasNext()) {
			Instruction i = instrIter.next();
			for (PcodeOp pcode : i.getPcode()) {
				pcodeStr += (i.getAddressString(false, true) + ":\t" + pcode.toString() + "\n");
			}
		}
		
		return pcodeStr;
	}

}
```
Here I just created a utility class that I'll add to in future posts that will grab the data for the different levels of PCode. I'll likely change this to a basic block-based approach in order to make graph rendering easier down the road, but for now a string will do. Here's the working view:

![A basic PCode view](/assets/images/pcode_basic.png)

Check out the [PCodeViewer plugin](https://github.com/bdemick/PCodeViewer) repo for the full source. Stay tuned for part 2 where I'll fumble around trying to make a graph-based view!

## Resources
[GhidraEmu](https://github.com/Nalen98/GhidraEmu) - A PCode emulation plugin with lots of GUI elements

[Kaiju](https://github.com/cmu-sei/kaiju) - A pretty extensive plugin that adds some pretty cool features, to include symbolic execution

[Ghidra Software Reverse Engineering for Beginners](https://www.packtpub.com/product/ghidra-software-reverse-engineering-for-beginners/9781800207974) - This book has a chapter on plugin development that helped demystify some of Ghidra's plugin plumbing
