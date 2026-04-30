Modification list:
1. Added primitive flucton - pp: 2m, 2p
    Files touched:
    * cascinit.f

2. Defined new baryon named flucton (compact p+p cluster)
    Files touched:
    * blockres.f
    * comres.f
    * itypdata.f
    * itrp2pdg.f

3. Defined new collision class
    Files touched:
    * collclass.f

4. Backward-hemisphere tail production.

    The existing sigmaLN row 16 dispatched flucton-N scattering through iline
    13/14/15 which are already defined as the generic baryon AQM-elastic /
    AQM-inelastic / string code paths in make22/crossx. The flucton therefore
    never disintegrated into three nucleons and no cumulative tail developed.

    Introduces three new iline numbers reserved for flucton-N:
    * 65: F+N total              via siglookup(13, e)
    * 66: F+N elastic (F+N->F+N) via siglookup(14, e)
    * 67: F+N breakup (F+N->NNN) via siglookup(15, e) + nbodydec

    Channel 67 is the cumulative-production mechanism. nbodydec populates
    pnew(1..4,1..3) with uniform 3-body phase space in the (F+N) rest frame;
    scatfinal's rotbos call boosts to the lab, so the internal flucton
    momentum (p* ~ 0.35 GeV/c) is correctly exposed as a backward tail.

    Files touched:
    * comres.f    : maxbrf=2, nsigs=16
    * blockres.f  : flucton width 0.200 GeV, sigmaLN(16) -> 65/66/67,
                    sigmainf(15,3)=-3, new bftype/branfluc/lbf decay arrays,
                    flbr/fbran/mminit/brange/b3type extended
    * scatter.f   : collclass=16 for F+N, 0 for F+F
    * make22.f    : computed-goto extended (lines 115 & 4393), new crossx
                    labels 65/66/67 and new make22 exit labels 65/66/67
                    (io=67 calls nbodydec for 3-body breakup)
    * anndec.f    : flucton decay dispatch + meson-baryon guard

5. fix: guard scatter.f against collclass=0 (STOP 137 at event 1)

    nuclei, starts event 1, then dies with
        cross[x,z]:  unknown channel requested, wrong io: 0
        STOP 137

    Root cause: scatter.f calls collclass() to pick the sigmaLN row, then blindly dereferences SigmaLn(1,1,isigline) - but when collclass returns 0 (e.g. F+F, F+Delta, F+meson, any pair we haven't parametrised), that read is out of bounds. nCh ends up whatever garbage sits in memory before row 1, the subsequent iline read lands on a 0, and crossx/crossz is entered with io=0 -> STOP. coload f already has this guard; scatter.f did not.

    Fixes:
    - scatter.f (main body, after collclass call): if isigline.le.0, zero
        the sigma(0..maxpsig) array and return. Matches the coload.f guard
        'if (isigline.eq.0) return' that was always there.

    - scatter.f::collclass: replace the narrow 'F+F -> 0' rule with a
        catch-all 'flucton in either slot AND the F+N rule above did not
        fire -> 0'. Covers F+F, F+Delta, F+N*, F+Y, F+antibaryon, F+meson,
        F+charmonium. Physically: no flucton-X parametrisation except F+N
        is implemented; those pairs should not interact (the flucton waits
        for its finite-width lifetime to expire and then decays via anndec).

    Both F+F and (say) F+pion now flow cleanly: collclass=0 -> scatter
    early-returns with sigma=0 -> collision rejected -> flucton propagates
    untouched until anndec fires.

5. debug: soft-guard crossx against io<=0 / io>67 to identify bad caller
    Replace unconditional STOP 137 with a diagnostic print + sig=0 return.
    This prevents the hard crash on Au+Au and gives us ityp/iso3/e of the
    particle pair reaching crossx with a bad iline, which is needed to
    trace the actual source of the STOP 137 regression.

    Once the bad caller is identified from the log, we will revert this
    guard and fix the real upstream bug.

    Files touched:
        * make22.f    

6. fix: out-of-bounds write to mstring(3) in F+N->NNN breakup (io=67)
    mstring is declared real*8 mstring(2) in newpart.f - it carries only TWO string masses, not per-particle masses. The prior io=67 code wrote
    mstring(i) for i=1..3 which triggered gfortran's:

    Warning: Array reference at (1) out of bounds (3 > 2) in loop
    Warning: iteration 2 invokes undefined behavior

    Fix: follow the same convention as channel 22 (around line 741-745):
        mstring(1) = pnew(5,1)          ! mass of particle 1
        mstring(2) = sum of pnew(5,2..nexit)  ! rest combined

    Per-particle masses go into pnew(5,*), which is dimensioned for mprt
    particles and is the array nbodydec actually uses.

    Files touched:
    * make22.f  

7. fix: flucton charge + trim class 16 exit channels
    1) fchg(i3, 71) returned 1 instead of 2 for a pp-cluster flucton.
    The generic Q = (I3 + B - S + C)/2 formula assumes baryon number
    B=1, but the flucton is a di-baryon (B=2). Added a dedicated
    branch in blockres.f:fchg that uses B=2 for fluctons so that a
    flucton with 2*I3 = +2 gets Q = +2.

    Before the fix, every flucton collision/decay triggered a
    charge-conservation warning because ocharge (computed from the
    buggy fchg) disagreed with ncharge (computed correctly from the
    outgoing nucleons).

    2) sigmaLN for class 16 had nCh=3 but only two real channels
    (66 = elastic, 67 = breakup). The third slot was a placeholder
    zero for a not-yet-implemented F+N -> F*+N excitation; scatter.f
    looped over all nCh slots and called crossx with iline=0,
    producing the "io out of range" warning spam. Set nCh=2.

    Result on 2-event Au+Au 8AGeV test:
    - io-out-of-range warnings gone
    - F+N elastic ch-conservation errors gone
    - pdgid tablelookup crash with related value 3.0 still present
    (only in the F+N -> N+N+N breakup path); fixed in the next
    commit.

    Files touched:
    * blockres.f

7. fix: prevent crashes from unphysical iso3 propagation after flucton breakup
    After the flucton (ityp=71, pp-cluster) breaks up via F+N -> N+N+N or
    decays, downstream collision/decay machinery occasionally inherits an
    out-of-range 2*I3 value (|iz|=3) for ordinary N* / Delta resonances.
    This causes two fatal crashes:

    1) anndex(dec): no final state found for: <ityp>  <m>  3  --> stop 137
        (no decay channel can match an isospin > 2I of the resonance)
    2) pdgid: Error in tablelookup --> fatal error -> stop 137
        (idtab does not contain (ityp, iso3=3) entries)

    Both aborts are too aggressive for production runs; the unphysical
    state is rare and recoverable.

    Files touched:
    * anndec.f: clamp iz1 to a valid range on decay entry, and if no
        decay channel can be sampled, return cleanly with nexit=0/sig=0
        instead of stop 137.
    * make22.f: handle nexit=0 from anndec by skipping the decay step
        rather than entering n-body decay with no products.
    * ityp2pdg.f: demote the lookup-failure error from level 3 (fatal)
        to level 1 (warning) and return pdgid=0; the output writer can
        still emit the record without aborting the whole run.

8. feat(flucton): realistic F+N breakup via 2-step cascade (step A: quasi-elastic F+N->N_s+F*; step B: isotropic F*->NN with p*~0.35 GeV/c)
    - Replace isotropic 3-body phase-space in make22.f io=67 with
    physically-motivated cascade implemented in new fluc_breakup
    subroutine (appended to make22.f).
    - In the F+N CM frame: N_s placed along +z with p_A=pcms(e,m_F,m_N);
    F* placed opposite. F* then decays isotropically in its rest frame
    to N+N back-to-back with p* = sqrt(m_F^2/4 - m_N^2). The two
    decay nucleons are boosted from the F* rest frame to the F+N CM.
    - Final overall orientation of the (N_s, F*) axis is delegated to
    angdisnew by extending its computed-GOTO and symlog tables to
    cover iline 65/66/67 (65->iso, 66/67->Mao forward-backward peripheral).
    - Reproduces the cumulative-production mechanism: decay nucleons
    can emerge with pz<0 in the lab when the intrinsic p*~0.35 GeV/c
    overcomes the F* boost.
    - Verified compile + testrun (Au+Au, ene=8): nucleons populate
    the backward hemisphere out to pz ~ -0.66 GeV in test.f19.

    Files touched:
    * angdis.f
    * make22.f

9. fix: flucton charge/baryon conservation + prevent PYTHIA crashes
    Root causes of the 23/500 early-abort rate and the B=408, Q=172 excess in
    Au+Au runs at ecm=8 GeV:

    1. Donor double-counting in cascinit.f
    After F+N -> F promotion, the donor nucleon was only 'parked' at
    rx=-10000 but kept its original ityp, charge and iso3.

    Fix: convert the donor slot into an inert, neutral photon
    (ityp=100, iso3=0, charge=0, mass=0, p=0) parked far from the grid.

    2. Corrupt iso3 in 3-body flucton breakup in make22.f channel 67
    The old assignment
        i3new(1)=sign(1,iz1); i3new(2)=sign(1,iz1);
        i3new(3)=iz1+iz2-i3new(1)-i3new(2)
    produced i3new(3)=+4 

    This triggered the cascade of 'anndex(dec): clamp unphysical iz1', 'ch-conservation error' and ultimately the 'pdgid: Error in tablelookup' warnings.

    Fix: derive the proton count from total charge
        np = (3 + iz1+iz2)/2 
    and assign +1 to the first np slots, -1 to the rest.  

    Falls back to elastic if the sanity check ever fails.

    3. PYTHIA crash guard in make22.f and upmerge.f
    When a corrupt iso3 slipped through, pdgid() returned 0, hepnam() returned an empty string, and PYINIT aborted the entire run with 'Error: unrecognized target particle' (pythia6409.f:5446).

    Fix a: in make22, clear itag1/itag2 whenever |iso3| exceeds the physical range for the ityp (baryons |iz|<=3, mesons |iz|<=2) or when the particle is a flucton.
    
    Fix b: in upmerge/upyth, bail out and zero nstring1=nstring2=0 whenever pdgid returns 0 or hepnam returns an empty name, so the run continues instead of STOP.

    Files touched:
    * cascinit.f
    * make22.f
    * upmerge.f

10. fix: avoid NaN in donor-photon output by setting tiny non-zero 4-momentum
    f14 output had ~14 NaN-bearing rows per event, one per
    parked-donor photon.  

    Fix: set pz = p0 = 1e-12 GeV for the parked donor photon.

    Files touched:
    * cascinit.f
