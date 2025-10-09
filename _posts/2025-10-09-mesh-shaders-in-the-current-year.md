# It Happened.

Just a quick post to confirm that the OpenGL/ES Working Group has signed off on the release of [GL_EXT_mesh_shader](https://github.com/KhronosGroup/OpenGL-Registry/pull/640).

# Credits
This is a monumental release, the largest extension shipped for GL this decade, and the culmination of many, many months of work by AMD. In particular we all need to thank Qiang Yu (AMD), who spearheaded this initiative and did the vast majority of the work both in writing the specification and doing the core mesa implementation. Shihao Wang (AMD) took on the difficult task of writing actual CTS cases (not mandatory for EXT extensions in GL, so this is a huge benefit to the ecosystem).

Big thanks to both of you, and everyone else behind the scenes at AMD, for making this happen.

Also we have to thank the [nvidium](https://github.com/MCRcortex/nvidium) project and its author, Cortex, for single-handedly pushing the industry forward through the power of Minecraft modding. Stay sane out there.

# Support
Minecraft mod support is already underway, so expect that to happen "soon".

The bones of this extension have already been merged into mesa over the past couple months. I opened a [MR to enable zink support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/37788) this morning since I have already merged the implementation.

Currently, I'm planning to wait until either just before the branch point next week or until RadeonSI merges its support to merge the zink MR. This is out of respect: Qiang Yu did a huge lift for everyone here, and ideally AMD's driver should be the first to be able to advertise that extension to reflect that. But the branchpoint is coming up in a week, and SGC will be going into hibernation at the end of the month until 2026, so this offer does have an expiration date.

In any case, we're done here.
